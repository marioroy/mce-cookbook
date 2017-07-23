## Many-Core Engine for Perl - Cookbook

This is a cookbook for demonstrating 
[MCE](https://metacpan.org/pod/MCE),
[MCE::Hobo](https://metacpan.org/pod/MCE::Hobo), and
[MCE::Shared](https://metacpan.org/pod/MCE::Shared). See also
[mce-examples](https://github.com/marioroy/mce-examples) for more recipes.

 - [Making an executable via PAR::Packer](#cross-platform-template-for-making-a-binary-executable-via-parpacker)
 - [Parallel-IO Reader with BioUtil::Seq](#parallel-io-reader-with-bioutilseq)
 - [Sharing Perl-Data-Language (PDL) on UNIX](#sharing-perl-data-language-pdl-on-unix)
 - [Sharing Perl-Data-Language (PDL) on Windows](#sharing-perl-data-language-pdl-on-windows)
 - [Sharing Perl-Data-Language (PDL) using threads](#sharing-perl-data-language-pdl-using-threads)
 - [Copyright and Licensing](#copyright-and-licensing)

------

### Cross-platform template for making a binary executable via PAR::Packer

Making an executable is possible with the L<PAR::Packer> module.
On the Windows platform, threads, threads::shared, and exiting via
threads are necessary for the binary to exit successfully.

```perl
 # https://metacpan.org/pod/PAR::Packer
 # https://metacpan.org/pod/pp
 #
 #   pp -o demo.exe demo.pl
 #   ./demo.exe

 use strict;
 use warnings;

 use if $^O eq "MSWin32", "threads";
 use if $^O eq "MSWin32", "threads::shared";

 use Time::HiRes (); # include minimum dependencies for MCE
 use Storable ();

 use IO::FDPass ();  # optional, for MCE::Shared->condvar, handle, queue
 use Sereal ();      # optional, faster serialization, may omit Storable

 use MCE;

 my $mce = MCE->new(
    max_workers => 4,
    user_func => sub {
       print "hello there from ", MCE->wid(), "\n";
    }
 );

 $mce->run();

 threads->exit(0) if $INC{"threads.pm"};
```

With MCE::Shared 1.808 and later releases, MCE::Hobo works just the
same. The following compiles fine on UNIX and the Windows platform.

```perl
 # https://metacpan.org/pod/PAR::Packer
 # https://metacpan.org/pod/pp
 #
 #   pp -o demo.exe demo.pl
 #   ./demo.exe

 use strict;
 use warnings;

 use if $^O eq "MSWin32", "threads";
 use if $^O eq "MSWin32", "threads::shared";

 use Time::HiRes (); # include minimum dependencies for MCE::Shared
 use Storable ();

 use IO::FDPass ();  # optional, for MCE::Shared->condvar, handle, queue
 use Sereal ();      # optional, faster serialization, may omit Storable

 use MCE::Hobo;      # 1.808 or later on Windows
 use MCE::Shared;

 my $seq_a = MCE::Shared->sequence( 1, 30 );

 sub task {
    my ( $id ) = @_;
    while ( defined ( my $num = $seq_a->next ) ) {
       print "$id: $num\n";
    }
 }

 MCE::Hobo->new( \&task, $_ ) for 1 .. 2;
 MCE::Hobo->waitall;

 threads->exit(0) if $INC{"threads.pm"};
```

### Parallel-IO Reader with BioUtil::Seq

MCE::Shared provides a "real" shared handle. Thus, allowing for parallel IO
iteration between many workers simultaneously.

This demonstration requires MCE 1.8xx or later to work.

```perl
 use strict;
 use warnings;

 use MCE::Hobo;
 use MCE::Shared;

 sub FastaReader2 {
    my ( $file, $not_trim ) = @_;

    my ( $open_flg ) = ( 0 );
    my ( $fh, $pos, $head );

    if ( $file =~ /^STDIN$/i ) {
       # from stdin
       $fh = MCE::Shared->handle( "<", \*STDIN );
    }
    elsif ( ref $file eq '' or ref $file eq 'SCALAR' ) {
       # from file
       $fh = MCE::Shared->handle( "<", $file )
          or die "fail to open file: $file!\n";
       $open_flg = 1;
    }
    else {
       # glob, i.e. given file handler
       # $fh = $file  # ok for MCE::Shared{ ->handle or ::Handle->new }
       #                                     (shared)    (non-shared)
       #
       # Below logic will not work if not a MCE::Shared handle. The reason
       # is that MCE::Shared/Handle treats "\n>" auto-magically; providing
       # records beginning with ">" and ending with "\n".

       return sub { return };
    }

    return sub {
       local $/ = "\n>";  # set input record separator

       while ( <$fh> ) {
          # skip comment section at the top of the file
          next if substr($_, 0, 1) ne '>';

          # extract header and sequence
          $pos  = index( $_, "\n" ) + 1;
          $head = substr( $_, 1, $pos - 2 );

          # $_ becomes sequence after substr
          substr( $_, 0, $pos, '' );

          # trim trailing "\r" in header
          chop $head if substr( $head, -1, 1 ) eq "\r";

          if ( length $head > 0 ) {
             $_ =~ tr/\t\r\n //d unless $not_trim;
             return [ $head, $_ ];
          }
       }

       close $fh if $open_flg;

       return;
    };
 }

 sub parallel_reader {
    my ( $id, $next_seq ) = @_;

    while ( my $fa = &$next_seq() ) {
       my ( $header, $seq ) = @$fa;
       print "$id: >$header\n$seq\n";

       sleep 1;  # simulate work
    }
 }

 #my $next_seq = FastaReader2("STDIN");
  my $next_seq = FastaReader2("sample.fasta");

  MCE::Hobo->new( \&parallel_reader, $_, $next_seq ) for 1 .. 3;

 #... do other stuff ...

  $_->join() for MCE::Hobo->list();
```

### Sharing Perl-Data-Language (PDL) on UNIX

Sharing PDL objects is possible with MCE 1.8 and above. Construction takes
place under the shared-manager process. PDL methods are processed automatically
through Perl's AUTOLOAD feature, inside MCE::Shared::Object.

```perl
 use strict;
 use warnings;

 use PDL;  # must load PDL before MCE

 use MCE::Hobo;
 use MCE::Shared;
 use Time::HiRes qw( time );

 my $tam     = @ARGV ? shift : 512;
 my $N_procs = @ARGV ? shift :   4;

 die "error: $tam must be an integer greater than 1.\n"
   if $tam !~ /^\d+$/ or $tam < 2;

 my ( $step_size   ) = ( ($tam >= 2048) ? 256 : ($tam >= 1024) ? 128 : 64 );
 my ( $cols, $rows ) = ( $tam, $tam );

 my $s = MCE::Shared->num_sequence( 0, $rows - 1, $step_size );
 my $o = MCE::Shared->pdl_zeroes( $rows, $rows );

 my $l = sequence( $cols, $rows );
 my $r = sequence( $rows, $cols );

 sub parallel_matmult {
    my ( $id ) = @_;

    while ( defined ( my $seq_n = $s->next() ) ) {
       my $start  = $seq_n;
       my $stop   = $start + $step_size - 1;
          $stop   = $rows - 1 if $stop >= $rows;

       my $result = $l->slice( ":,$start:$stop" ) x $r;

     # --- action taken by the shared-manager process
     # ins_inplace(  1 arg  ):  ins( inplace( $this ), $what, 0, 0 );
     # ins_inplace(  2 args ):  $this->slice( $arg1 ) .= $arg2;
     # ins_inplace( >2 args ):  ins( inplace( $this ), $what, @coords );

     # -- use case
     # $o->ins_inplace( $result );                    #  1 arg
     # $o->ins_inplace( ":,$start:$stop", $result );  #  2 args
       $o->ins_inplace( $result, 0, $seq_n );         # >2 args
    }

    return;
 }

 my $start = time;

 MCE::Hobo->create( \&parallel_matmult, $_ ) for 1 .. $N_procs;

 # ... do other stuff ...

 $_->join() for MCE::Hobo->list();

 printf "\ntam $tam, duration: %0.03f secs\n\n", time - $start;

 # export/destroy the PDL object from shared_process if desired
 # $o = $o->destroy();

 # Output logic is consistent with example by David Mertens.
 # https://gist.github.com/run4flat/4942132

 for my $pair ( [0, 0], [324, 5], [42, 172], [$tam-1, $tam-1] ) {
    my ( $row, $col ) = @$pair;
    $row %= $rows, $col %= $cols;
    print "( $row, $col ): ", $o->at( $row, $col ), "\n";
 }

 print "\n";
```

### Sharing Perl-Data-Language (PDL) on Windows

Unfortunately, the above example will not work on the Windows platform.
Therefore, all 3 matrices must be shared. Workers make a local copy for
the right matrix only. Another possibility is using memory mapped data;
see matmult/matmult_mce_d.pl, included with mce-examples on Github.

The next section provides a demonstration using PDL::Parallel::threads.

```perl
 my $o = MCE::Shared->pdl_zeroes( $rows, $rows );

 my $l = MCE::Shared->pdl_sequence( $cols, $rows );
 my $r = MCE::Shared->pdl_sequence( $rows, $cols );

 sub parallel_matmult {
    my ( $id ) = @_;
    my $r_copy = $r->sever;

    while ( defined ( my $seq_n = $s->next() ) ) {
       my $start  = $seq_n;
       my $stop   = $start + $step_size - 1;
          $stop   = $rows - 1 if $stop >= $rows;

       my $result = $l->slice( ":,$start:$stop" ) x $r_copy;

     # --- action taken by the shared-manager process
     # ins_inplace(  1 arg  ):  ins( inplace( $this ), $what, 0, 0 );
     # ins_inplace(  2 args ):  $this->slice( $arg1 ) .= $arg2;
     # ins_inplace( >2 args ):  ins( inplace( $this ), $what, @coords );

     # -- use case
     # $o->ins_inplace( $result );                    #  1 arg
     # $o->ins_inplace( ":,$start:$stop", $result );  #  2 args
       $o->ins_inplace( $result, 0, $seq_n );         # >2 args
    }

    return;
 }
```

### Sharing Perl-Data-Language (PDL) using threads

The prior example consumes unnecessary memory consumption. Fortunately,
there's another way via PDL::Parallel::threads. This requires Perl to be
built with threads support.

```perl
 use strict;
 use warnings;

 use PDL;
 use PDL::Parallel::threads qw(retrieve_pdls);

 use threads;
 use MCE::Shared;
 use Time::HiRes qw( time );

 my $tam     = @ARGV ? shift : 512;
 my $N_procs = @ARGV ? shift :   4;

 die "error: $tam must be an integer greater than 1.\n"
   if $tam !~ /^\d+$/ or $tam < 2;

 my ( $step_size   ) = ( ($tam >= 2048) ? 256 : ($tam >= 1024) ? 128 : 64 );
 my ( $cols, $rows ) = ( $tam, $tam );

 my $s = MCE::Shared->num_sequence( 0, $rows - 1, $step_size );

 my $o = zeroes( $rows, $rows );     $o->share_as('output');
 my $l = sequence( $cols, $rows );   $l->share_as('left_input');
 my $r = sequence( $rows, $cols );   $r->share_as('right_input');

 sub parallel_matmult {
    my ( $id ) = @_;
    my ( $o, $l, $r ) = retrieve_pdls('output', 'left_input', 'right_input');

    while ( defined ( my $seq_n = $s->next() ) ) {
       my $start  = $seq_n;
       my $stop   = $start + $step_size - 1;
          $stop   = $rows - 1 if $stop >= $rows;

       my $result = $l->slice( ":,$start:$stop" ) x $r;

     # ins( inplace($o), $result, 0, $seq_n );
       $o->slice(":,$start:$stop") .= $result;

     # use PDL::NiceSlice;
     # $o(:,$start:$stop) .= $result;
     # no  PDL::NiceSlice;
    }

    return;
 }

 my $start = time;

 threads->create( \&parallel_matmult, $_ ) for 1 .. $N_procs;

 # ... do other stuff ...

 $_->join() for threads->list();

 printf "\ntam $tam, duration: %0.03f secs\n\n", time - $start;

 # Output logic is consistent with example by David Mertens.
 # https://gist.github.com/run4flat/4942132

 for my $pair ( [0, 0], [324, 5], [42, 172], [$tam-1, $tam-1] ) {
    my ( $row, $col ) = @$pair;
    $row %= $rows, $col %= $cols;
    print "( $row, $col ): ", $o->at( $row, $col ), "\n";
 }

 print "\n";
```

### Copyright and Licensing

Copyright (C) 2012-2017 by Mario E. Roy <marioeroy AT gmail DOT com>

This program is free software; you can redistribute it and/or modify
it under the same terms as Perl itself:

        a) the GNU General Public License as published by the Free
        Software Foundation; either version 1, or (at your option) any
        later version, or

        b) the "Artistic License" which comes with this Kit.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See either
the GNU General Public License or the Artistic License for more details.

You should have received a copy of the Artistic License with this
Kit, in the file named "Artistic".  If not, I'll be glad to provide one.

You should also have received a copy of the GNU General Public License
along with this program in the file named "Copying". If not, write to the
Free Software Foundation, Inc., 51 Franklin Street, Fifth Floor,
Boston, MA 02110-1301, USA or visit their web page on the internet at
http://www.gnu.org/copyleft/gpl.html.

