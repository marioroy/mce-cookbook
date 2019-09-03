## Many-Core Engine for Perl - Cookbook

This is a cookbook for demonstrating 
[MCE](https://metacpan.org/pod/MCE),
[MCE::Hobo](https://metacpan.org/pod/MCE::Hobo), and
[MCE::Shared](https://metacpan.org/pod/MCE::Shared). See also
[mce-examples](https://github.com/marioroy/mce-examples) for more recipes.

 - [Making an executable via PAR::Packer](#cross-platform-template-for-making-a-binary-executable-via-parpacker)
 - [Parallel-IO Reader for BioUtil::Seq](#parallel-io-reader-for-bioutilseq)
 - [Parallel-IO Reader via Bio::SeqIO](#parallel-io-reader-via-bioseqio)
 - [Parallel-IO Bio::SeqIO reformatter](#parallel-io-bioseqio-reformatter)
 - [Sharing Perl-Data-Language (PDL)](#sharing-perl-data-language-pdl)
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

 # Include minimum dependencies for MCE.
 # Add other modules required by your application here.

 use Storable ();
 use Time::HiRes ();

 # use Sereal ();     # optional, for faster serialization

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

MCE workers above spawn as threads whenever threads is present. Unlike MCE,
MCE::Hobo workers can only run as child processes, spawned via fork. To work
around PAR or a dependency not being multi-process safe, one must run inside
a thread on the Windows platform or the exe will crash.

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

 # Include minimum dependencies for MCE::Hobo.
 # Add other modules required by your application here.

 use Storable ();
 use Time::HiRes ();

 # use IO::FDPass ();  # optional: for condvar, handle, queue
 # use Sereal ();      # optional: for faster serialization

 use MCE::Hobo;
 use MCE::Shared;

 # For PAR to work on the Windows platform, one must include manually
 # any shared modules used by the application.

 # use MCE::Shared::Array;    # for MCE::Shared->array
 # use MCE::Shared::Cache;    # for MCE::Shared->cache
 # use MCE::Shared::Condvar;  # for MCE::Shared->condvar
 # use MCE::Shared::Handle;   # for MCE::Shared->handle, mce_open
 # use MCE::Shared::Hash;     # for MCE::Shared->hash
 # use MCE::Shared::Minidb;   # for MCE::Shared->minidb
 # use MCE::Shared::Ordhash;  # for MCE::Shared->ordhash
 # use MCE::Shared::Queue;    # for MCE::Shared->queue
 # use MCE::Shared::Scalar;   # for MCE::Shared->scalar

 # Et cetera. Only load modules needed for your application.

 use MCE::Shared::Sequence;   # for MCE::Shared->sequence

 my $seq = MCE::Shared->sequence( 1, 9 );

 sub task {
     my ( $id ) = @_;
     while ( defined ( my $num = $seq->next() ) ) {
         print "$id: $num\n";
         sleep 1;
     }
 }

 sub main {
     MCE::Hobo->new( \&task, $_ ) for 1 .. 3;
     MCE::Hobo->waitall();
 }

 # Main must run inside a thread on the Windows platform or workers
 # will fail duing exiting, causing the exe to crash. The reason is
 # that PAR or a dependency isn't multi-process safe.

 ( $^O eq "MSWin32" ) ? threads->create(\&main)->join() : main();

 threads->exit(0) if $INC{"threads.pm"};
```

### Parallel-IO Reader for BioUtil::Seq

MCE::Shared provides a "real" shared handle. Thus, allowing for parallel IO
iteration between many workers simultaneously. This demonstration requires
MCE::Shared 1.831 or later to run.

```perl
 use strict;
 use warnings;

 use MCE::Hobo;
 use MCE::Shared;

 sub FastaReader2 {
    my ( $file, $not_trim ) = @_;

    my ( $open_flg ) = ( 0 );
    my ( $fh, $pos, $hdr, $id, $desc );

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
       # glob: i.e. given file handler
       # $fh = $file  # ok for MCE::Shared{ ->handle or ::Handle->new }
       #                                     (shared)    (non-shared)
       #
       # Below logic will not work if not a MCE::Shared handle. The reason
       # is that MCE/Shared/Handle.pm treats "\n>" auto-magically; providing
       # records beginning with ">" and ending with "\n".

       return sub { return };
    }

    return sub {
       local $/ = "\n>";  # set input record separator

       while ( <$fh> ) {
          # skip comment section at the top of the file
          next if substr($_, 0, 1) ne '>';

          # extract header and sequence
          $pos = index( $_, "\n" ) + 1;
          $hdr = substr( $_, 1, $pos - 2 );

          # $_ becomes sequence after substr
          substr( $_, 0, $pos, '' );

          # trim trailing "\r" in header
          chop $hdr if substr( $hdr, -1, 1 ) eq "\r";

          # id and description
          ($id, $desc) = split(' ', $hdr, 2);
          $desc = '' unless defined $desc;

          if ( length $hdr > 0 ) {
             $_ =~ tr/\t\r\n //d unless $not_trim;
             return [ $id, $desc, $_ ];
          }
       }

       close $fh if $open_flg;

       return;
    };
 }

 sub parallel_reader {
    my ( $hoboID, $next_seq ) = @_;

    while ( my $fa = &$next_seq() ) {
       my ( $id, $desc, $seq ) = @$fa;
       print "$hoboID: >$id $desc\n$seq\n";

       sleep 1;  # simulate work
    }
 }

 my $next_seq = FastaReader2("sample.fasta");  # or "STDIN"

 for my $hoboID ( 1 .. 3 ) {
    MCE::Hobo->new( \&parallel_reader, $hoboID, $next_seq );
 }

 #... do other stuff ...

 MCE::Hobo->waitall();
```

### Parallel-IO Reader via Bio::SeqIO

A simplier demonstration using a shared Bio::SeqIO handle. Like the prior
example, MCE::Shared 1.831 or later is required.

```perl
 use strict;
 use warnings;

 use MCE::Hobo;
 use MCE::Shared;

 use Bio::Seq;
 use Bio::SeqIO;

 sub parallel_reader {
    my ( $hoboID, $seqIO ) = @_;

    while ( my $next = $seqIO->next_seq() ) {
       my ( $id, $desc, $seq ) = ( $next->id, $next->desc, $next->seq );
       print "$hoboID: >$id $desc\n$seq\n";

       sleep 1;  # simulate work
    }
 }

 my $seqIO = MCE::Shared->share( { module => 'Bio::SeqIO' },
    -file => "sample.fasta", -format => "Fasta"
 );

 for my $hoboID ( 1 .. 3 ) {
    MCE::Hobo->new( \&parallel_reader, $hoboID, $seqIO );
 }

 #... do other stuff ...

 MCE::Hobo->waitall();
```
 
### Parallel-IO Bio::SeqIO reformatter

The Bio::SeqIO module provides a simple reformatter example. Running parallel
is possible. Here, workers request the next sequence from the manager process.
The manager process also writes orderly to STDOUT.

Reading by MCE workers is record-driven when the RS option is specified.
Unfortunately, that will not work for the fastq format "\n\@" due to quality
data containing "@" at the start of line. A way around this is having workers
request the next sequence via an input iterator. A fasta iterator is provided
supporting Fasta and Fastq formats.


```perl
 use strict;
 use warnings;

 use MCE;
 use MCE::Candy;
 use Scalar::Util 'reftype';

 use Bio::SeqIO;
 use IO::String;

 my $format1 = shift;
 my $format2 = shift || die
    "Usage: perl reformat.pl format1 format2 < input > output\n",
    "       perl reformat.pl fasta   genbank < input > output\n",
    "       perl reformat.pl fasta   genbank   input > output (faster)\n\n";

 my %recsep = (
    embl    => "\nID",    fasta => "\n\>",   # start of record
    genbank => "\nLOCUS", fastq => "\n\@",
    swiss   => "\nID",    pir   => "\n\>",
 );

 die "Supported input format: ", join(", ", sort keys %recsep), "\n\n"
    unless exists $recsep{ lc $format1 };

 my $input_io  = IO::String->new( my $input  = '' );
 my $output_io = IO::String->new( my $output = '' );

 my $in  = Bio::SeqIO->newFh( -format => $format1, -fh => $input_io );
 my $out = Bio::SeqIO->newFh( -format => $format2, -fh => $output_io );

 die unless $in && $out;  # error/format cannot be found above

 # Workers request sequences via an input iterator.

 if ( lc $format1 eq 'fastq' ) {
    MCE->new(
       input_data => $ARGV[0] ? fasta_iter($ARGV[0],1) : fasta_iter(\*STDIN,1),

       max_workers => 'auto', chunk_size => 2,
       gather => MCE::Candy::out_iter_fh(\*STDOUT),

       user_func => sub {
          my ( $mce, $chunk_ref, $chunk_id ) = @_;

          # rewind/truncate strings
          $input_io->setpos(0), $output_io->setpos(0);
          $input = '', $output = '';

          # set input string
          for my $next_seq (@{ $chunk_ref }) {
             my ( $id, $desc, $seq, $qual ) = @{ $next_seq };
             $input .= "\@$id $desc\n$seq\n\+$id $desc\n$qual\n";
          }

          # read input, reformat into output
          print $out $_ while <$in>;

          # send string to the manager process
          MCE->gather( $chunk_id, $output );
       }
    )->run();
 }

 # Workers read/request raw text, input records determined by RS.

 else {
    MCE->new(
       RS => $recsep{ lc $format1 },
       input_data => $ARGV[0] ? $ARGV[0] : \*STDIN,

       max_workers => 'auto', chunk_size => 2,
       gather => MCE::Candy::out_iter_fh(\*STDOUT),

       user_func => sub {
          my ( $mce, $chunk_ref, $chunk_id ) = @_;

          # rewind/truncate strings
          $input_io->setpos(0), $output_io->setpos(0);
          $input = '', $output = '';

          # set input string
          for my $next_seq (@{ $chunk_ref }) {
             $input .= $next_seq;
          }

          # read input, reformat into output
          print $out $_ while <$in>;

          # send string to the manager process
          MCE->gather( $chunk_id, $output );
       }
    )->run();
 }

 # Bio-Fasta/Fastq record-driven input iterator with chunking support.
 # The chunk_size value is passed to the closure block when called by MCE.

 sub fasta_iter {
    my ( $file, $is_fastq ) = @_;
    my ( $off, $pos, $flag, $hdr, $id, $desc, $seq, $qual );
    my ( $fh, $finished, $chunk_size, @chunk );

    if ( !reftype $file || reftype $file ne 'GLOB' ) {
       open $fh, '<', $file or die "open error '$file': $!\n";
    } else {
       $fh = $file;
    }

    {
       local $/ = \1; my $byte = <$fh>;             # read one byte

       unless ( $byte eq ( $is_fastq ? '@' : '>' ) ) {
          $/ = $is_fastq ? "\n\@" : "\n\>";         # skip comment section
          my $skip_comment = <$fh>;                 # at the top of file
       }
    }

    return sub {
       return if $finished;
       local $/ = $is_fastq ? "\n\@" : "\n\>";      # input record separator

       $chunk_size = shift || 1;

       while ( $seq = <$fh> ) {
          if ( substr($seq, -1, 1) eq ( $is_fastq ? '@' : '>' ) ) {
              substr($seq, -1, 1, '');              # trim trailing @ or >
          }
          $pos = index($seq, "\n") + 1;             # header and sequence
          $hdr = substr($seq, 0, $pos - 1);         # extract the header, then
          substr($seq, 0, $pos, '');                # ltrim header from seq

          chop $hdr if substr($hdr, -1, 1) eq "\r"; # rtrim trailing "\r"
          ( $id, $desc ) = split(' ', $hdr, 2);     # id and description

          $desc = '' unless defined $desc;
          $flag = 0;

          if ( $is_fastq && ( $pos = index($seq, "\n+") ) > 0 ) {
             $off = length($seq) - $pos;
             if ( ( $pos = index($seq, "\n", $pos + 1) ) > 0 ) {
                $qual = substr($seq, $pos);         # extract quality
                $qual =~ tr/\t\r\n //d;             # trim white space
                $flag = 1;
             }
             substr($seq, -$off, $off, '');         # rtrim qual from seq
          }

          $seq =~ tr/\t\r\n //d;                    # trim white space

          if ( $flag && length($qual) != length($seq) ) {
             # extract quality until length matches sequence
             do {
                my $tmp = <$fh>; $tmp =~ tr/\t\r\n //d;
                substr($tmp, -1, 1, '') unless eof($fh);
                $qual .= '@'; $qual .= $tmp;
             } until ( length($qual) == length($seq) || eof($fh) );
          }

          ( $chunk_size > 1 )
             ? push @chunk, [ $id, $desc, $seq, $qual || '' ]
             : return       [ $id, $desc, $seq, $qual || '' ];

          return splice(@chunk, 0, $chunk_size)
             if ( @chunk == $chunk_size );
       }

       close $fh if ( fileno $fh > 0 );
       $finished = 1;

       return splice(@chunk, 0, scalar @chunk)
          if ( $chunk_size > 1 && @chunk );

       return;
    }
 }
```

### Sharing Perl-Data-Language (PDL)

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

 # On Windows, the ($l,$r) piddles are unblessed in worker threads.
 # Therefore, constructing ($l,$r) inside the worker versus sharing.
 # UNIX platforms benefit from copy-on-write. Thus, one copy.
 #
 # Results are stored in the shared piddle ($o) above.

 my $l = ( $^O eq 'MSWin32' ) ? undef : sequence( $cols, $rows );
 my $r = ( $^O eq 'MSWin32' ) ? undef : sequence( $rows, $cols );

 sub parallel_matmult {
    my ( $id ) = @_;

    $l = sequence( $cols, $rows ) unless ( defined $l );
    $r = sequence( $rows, $cols ) unless ( defined $r );

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

### Sharing Perl-Data-Language (PDL) using threads

The prior example may consume unnecessary memory consumption when using threads.
Fortunately, there is another way via PDL::Parallel::threads.

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

