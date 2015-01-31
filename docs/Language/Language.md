## The about statement

#### Synopsis

    
    
        about title: <title for pipeline>
    
  

#### Behavior

The *about* statement defines pipeline level documentation for a pipeline file.  It can be used any where at the top level of a pipeline file.  At the moment, *title* is the only supported attribute for pipeline documentation.

Pipeline documentation is used in the HTML report that can be generated using the [run](run) command.

#### Examples

**Add a title to a pipeline**
```groovy 

about title: "Exome Variant Calling Pipeline"

run { align_bwa + picard_dedupe + gatk_call_variants }
```
---
## Variables in Bpipe

Bpipe supports variables inside Bpipe scripts.  In Bpipe there are two kinds of variables: 

- *implicit* variables  - variables defined by Bpipe for you
- *explicit* variables  - variables you define yourself

#### Implicit Variables

Implicit variables are special variables that are made available to your Bpipe pipeline stages automatically.   The two important implicit variables are:

- input - defines the name(s) of files that are inputs to your pipeline stage
- output - defines the name(s) of files that are outputs from your pipeline stage
- environment variables - variables inherited from your shell environment (note: this feature is not implemented yet - coming soon!).
- branch - when a pipeline is running parallel branches, the name of the current branch is available in a *$branch* variable.

The input and output variables are how Bpipe automatically connects tasks together to make a pipeline.  The default input to a stage is the output from the previous stage. In general you should always try to use these variables instead of hard coding file names into your commands.   Using these variables ensures that your tasks are reusable and can be joined together to form flexible pipelines.  

#### =Extension Syntax for Input and Output Variables=

Bpipe provides a special syntax for easily referencing inputs and outpus with specific file extensions. See [[ExtensionSyntax]] for more information.

**Multiple Inputs**

Different tasks have different numbers of inputs and outputs, so what happens when a stage with multiple outputs is joined to a stage with only a single input?  Bpipe's goal is to try and make things work no matter what stages you join together.  To do this:

- The variable *input1* is always a single file name that a task can read inputs from.  If there were multiple outputs from the previous stage, the *input1* variable is the first of those outputs.  This output is treated as the "primary" or "default" output.  The variable `$input` also evaluates, by default, to the value of *input1*.
- If a task wants to accept more than one input then it can reference each input using the variables *input1*, *input2*, etc.
- If a task has an unknown number of inputs it may reference the variable *input* as a *list* and use array like indices to access the elements, for example `_input[is the same as `input1`, `input[1](0]_`)` corresponds to `input2`, etc.   You can reference the size of the inputs using `input.size()` and iterate over inputs using a syntax like
```groovy 

  for(i in input) {
    exec "... some command that reads $i ..."
  }
```

- When a stage has multiple outputs, the first such output is treated as the "primary" output and appears to the next stage as the "input" variable.  If the following stage only accepts a single input then its input will be the primary output of the previous stage.

#### Explicit Variables

Explicit variables are ones you define yourself.  These variables are created inline inside your Bpipe scripts using Java-like (or Groovy) syntax.  They can be defined inside your tasks or outside of your tasks to share them between tasks.  For example, here two variables are defined and shared between two tasks:
```groovy 

  NUMTHREADS=8
  REFERENCE_GENOME="/data/hg19/hg19.fa"

  align_reads = {
    exec "bwa aln -t $NUMTHREADS $REFERENCE_GENOME $input"
  }
  
  call_variants = {
    exec "samtools mpileup -uf $REFERENCE_GENOME $input > $output"
  }
```

#### Variable Evaluation

Most times the way you will use variables is by referencing them inside shell commands that you are running using the *exec* statement.  Such statements define the command using single quotes, double quotes or triple quotes.  Each kind of quotes handles variable expansion slightly differently:

**Double quotes** cause variables to be expanded before they are passed to the shell.  So if input is "myfile.txt" the statement: 
```groovy 

  exec "echo $input"
```

will reach the shell as exactly:
```groovy 

  echo myfile.txt
```

Since the shell sees no quotes around "myfile.txt" this will fail if your file name contains spaces or other characters treated specially by the shell.  To handle that, you could embed single quotes around the file name:
```groovy 

  exec "echo '$input'"
```

**Single quotes** cause variables to be passed through to the shell without expansion.  Thus 
```groovy 

  exec 'echo $input'
```

will reach the shell as exactly:
```groovy 

  echo $input
```

**Triple quotes** are useful because they accept embedded newlines.  This allows you to format long commands across multiple lines without laborious escaping of newlines.   Triple quotes escape variables in the same way as single quotes, but they allow you to embed quotes in your commands which are passed through to the shell.  Hence another way to solve the problem of spaces above would be to write the statement as:
```groovy 

  exec """
    echo "$input"
  """
```

See the [[Exec|exec]] statement for a longer example of using triple quotes.

#### Referencing Variables Directly

Inside a task the variables can be referenced using Java-like (actually Groovy) syntax.  In this example Java code is used to check if the input already exists:
```groovy 

  mytask = {
      // Groovy / Java code!
      if(new File(input).exists()) {
        println("File $input already exists!")
      }
  }
```

You won't normally need to use this kind of syntax in your Bpipe scripts, but it is available if you need it to handle complicated or advanced scenarios.

#### Differences from Bash Variable Syntax

Bpipe's variable syntax is mostly compatible with the same syntax in languages like Bash and Perl. This is very convenient because it means that you can copy and paste commands directly from your command line into your Bpipe scripts, even if they use environment variables.

However there are some small differences between Bpipe variable syntax and Bash variable syntax:

- Bpipe always expects a `$` sign to be followed by a variable name.  Thus operations in Bash that use `$` for other things need to have the `$` escaped so that Bpipe does not interpret it.  For example:

```groovy 

  exec "for i in $(ls **.bam); do samtools index $i; done"
```

In this case the $ followed by the open bracket is illegal because Bpipe will try to interpret it as a variable.  We can fix this with a backslash:
```groovy 

  exec "for i in \$(ls **.bam); do samtools index $i; done"
```

- Bpipe treats a `.` as a special character for querying a property, while Bash merely delimits a variable name with it. Hence if you wish to write `$foo.txt` to have a file with '.txt' appended to the value of variable foo, you need to use curly braces: `${foo}.txt`.

---
## Branch Variables

A common need is to communicate information between disparate parts of your pipeline. Often this information is scoped to the context of a particular branch or segment of your pipeline. Bpipe supports this by the use of "branch variables".

#### Creating Branch Variables

Each branch of a pipeline (including the root), is passed an implicit variable called 'branch'. If referenced as a value this variable evaluates to the name of the branch in which the stage is running. For example, if you have split by chromosome using the [[Chr|chr]] command, the name of the branch is the chromosome for which the branch is executing. If you have split by file then it is the name of the file (or unique portion thereof) that is specific to the branch.

The `branch` variable, however, also acts as a container for your own properties. Thus you can set properties on it using normal Groovy code:

```groovy 

hello = {
    branch.planet = "mars"
}
```

#### Referencing Branch Variables

Once set, a branch variable becomes referenceable directly by all subsequent stages that are part of the branch, including any child branches. The value can be referenced as a property on the `branch` object, for example '`branch.planet`', but can also be referenced without the `branch.` prefix.  For example, the following pipeline prints "hello mars":

```groovy 

hello = {
    branch.planet = "mars"
}

world = {
    exec """echo "Hello $planet" """
}

run { hello + world }
```
---
## The check statement

#### Synopsis

    
    
      check {
         < statements to validate results >
      } otherwise {
          <statements to execute on failure >
      }
    
  
    
    
      check(<check name>) {
         < statements to validate results >
      } otherwise {
          <statements to execute on failure >
      }
    

#### Availability

0.9.8.6_beta_2 +

#### Behavior

The *check* statement gives a convenient way to implement validation of a pipeline's outputs or progress and implement an alternative action if the validation fails. The `check` clause is executed and any `exec` or other statements inside are processed. If one of these fails, then the `otherwise` clause executes.

The *check* statement is stateful. Bpipe remembers the result and does not re-execute a check unless the input files are updated. Thus it is possible to implement long-running, intensive tasks to perform checks as just like normal Bpipe commands, they will not be re-executed if the pipeline is re-executed. The state is remembered by files that are created in the `.bpipe/checks` directory in the pipeline directory. Effectively, the created check file is treated as an implicit output of the `check` clause.

A convenient use of `check` is in conjunction with [[Success|success]], [[Fail|fail]] and [[Send|send]] commands.

*Note*:  a check does not have to result in aborting of the pipeline.  You may choose to do nothing in the otherwise clause of the check (it must still exist though), in which case the check is merely informational. Alternatively, the `succeed` command will cause the current branch to terminate and not produce any output files, but leave other branches running. To fail the whole pipeline, use the `fail` command.

*Note 2*: due to a quirk of Groovy syntax, the *otherwise* command **must** be placed on the same line as the preceding curly bracket of the `check` clause.

#### Examples

**Check that output file is non-zero length and fail the whole pipeline if it is not**
```groovy 

  check {
        exec "[ -s $output ]"
  } otherwise {
        fail "The output file had zero length"
  }
```

**Check that output file is non-zero length and terminate only this branch if it is not**
```groovy 

  check {
        exec "[ -s $output ]"
  } otherwise {
        succeed "The output file had zero length"
  }
```

**Check that output file is non-zero length and notify by email if it is not**
```groovy 

  check {
        exec "[ -s $output ]"
  } otherwise {
        send "Output file $output had zero size" to gmail
  }
```
---
## The chr statement

#### Synopsis

    
    
      chr(<digit|letter> sequence, <digit|letter> sequence) * [ <stage1> + <stage2> + ..., ... ]
    

#### Behavior

The `chr` statement splits execution into parallel paths based on chromosome. For the general case, all the files supplied as input are forwarded to every set of parallel stages supplied in the following list. (See below for an exception to this).

The stages that are executed in parallel receive a special implicit variable, 'chr' that can be used to reference the chromosome that should be processed. This is typically passed through to tools that can confine their operations to a genomic region as an argument.

*Note*: Bpipe doesn't do anything to split the input data by chromosome: all the receiving stages get the whole data set. Thus for this to work, the tools that are executed need to support operations on genomic subregions.

#### = File Naming Conventions =

When executing inside a chromosomal specific parallel branch of the pipeline, output files are automatically created with the name of the chromosome embedded. This ensures that different parallel branches do not try to write to the same output file. For example, instead of an output file being named "hello.txt" it would be named "hello.chr1.txt" if it was in a parallel branch processing chromosome 1.

If files that are supplied as inputs contain chromosomal segments as part of their file name then the inputs are filtered so that only inputs containing the corresponding chromosome name are forwarded to the parallel segment containing that name. This behavior can be disabled by adding `filterInputs: false` as an additional argument to `chr`.

#### Examples

** Call variants on a bam file of human reads from each chromosome in parallel **
```groovy 

gatk_call_variants = {
      exec """
        java -Xmx5g -jar GenomeAnalysisTK.jar -T UnifiedGenotyper
            -R hg19.fa
            -D dbsnp_132.hg19.vcf
            -glm BOTH
            -I $input.bam
            -stand_call_conf 5
            -stand_emit_conf 5
            -o $output.vcf
            -metrics ${output}.metrics
            -L $chr
        """
    }
}

chr(1..22,'X','Y') * [ gatk_call_variants ]
```
---
## Directory Convenience Function

### Directory Convenience Function

Sometimes a command you want requires not the name of an output file, but the name of the directory in which that file resides. This can cause a bit of confusion to Bpipe, because the real output is the file, but what you reference in your command is the directory.

To help with this, Bpipe offers a special meaning for the "dir" extension. When you reference `$output.dir` Bpipe treats it as a reference to the output **file**, but it passes the directory in which the file resides to the command that is executing.

The `$input.dir` variable also has special meaning. In this case the `dir` is taken to mean that the whole directory itself should be considered an input, and Bpipe will search for an input that is in fact a directory, rather than a file. This enables you to use the file-extension metaphor for selecting directories within your pipeline for input to your commands. 

#### Setting the Output Directory

While the `input.dir` is a fixed value, `output.dir` is a writeable value, so you can use it to change the value of directory to which outputs will go. Bpipe currently expects all outputs from a pipeline stage to go to the same directory, so setting this to multiple values or different values will not work.

*Note*: if you modify `output.dir`, be mindful that it must execute before the expression defining the command you wish to execute is evaluated. For example, the following will not work, because output.dir is changed after the output.txt was already evaluated.

```groovy 

hello = {
   output.dir="out_dir"
   def myOutput = output.txt
   exec "cp $input.csv $myOutput"
}
```

#### Example

Here we wish to have fastqc put its output zip file into a directory called "qc_data". However fastqc won't let us tell it the full path of the zip file, only the directory name.
```groovy 

fastqc = {
    // Start by telling Bpipe where the outputs are going to go
    output.dir = "qc_data"

    // Then tell Bpipe how the file name is getting transformed
    // Notice we don't need any reference to the output directory here!
    transform(".fastq.gz") to ("_fastqc.zip") {
        // Now in our command, Bpipe gives us the folder as the value,
        // while knowing that it is referencing the zip file output 
        // specified in our transform
        exec "mkdir -p $output.dir; fastqc -o $output.dir $input.gz" 
    }
}
```
---
## The doc statement

#### Synopsis

    
    
      doc [title] | ( attribute1 : value, attribute2: value...)
    
  
#### Attributes

Valid documentation Attributes are:

<table>
  <tr><td>title</td><td>Brief one line description of the pipeline stage</td></tr>
  <tr><td>desc</td><td>Longer description of a pipeline stage</td></tr>
  <tr><td>author</td><td>Name or email address of the author of the pipeline stage</td></tr>
  <tr><td>constraints</td><td>Any warnings or constraints about using the pipeline stage</td></tr>
</table>

#### Behavior

A *doc* statement adds documentation to a pipeline stage.  It is only valid within a declaration of a pipeline stage.  The *doc* statement has one of two forms - a brief from that allows you to give a simple one line description of a pipeline stage and a longer form that lets you specify multiple attributes.

Documentation added with a *doc* statement is currently used when you generate a HTML report for a run (see [run](run) command).

*Note*: Bpipe will augment documentation provided with the `doc` command with additional values that are determined at run time, such as the inputs of the pipeline stage, the outputs of the pipeline stage, the versions of executable tools used in the pipeline stage (where those have been able to be determined) and the execution time details (start, stop, duration).

#### Examples

**Add a title for a pipeline stage**
```groovy 

  align_with_bwa = {
      doc "Align FastQ files using BWA"
  }
```

**Add full documentation for a pipeline stage**
```groovy 

  align_with_bwa = {
      doc title: "Align FastQ files using BWA",
          desc:  """Performs alignment using BWA on Illumina FASTQ 
                    input files, producing output BAM files.  The BAM 
                    file is not indexed, so you may need to follow this
                    stage with an "index_bam" stage.""",
          constraints: "Only works with Illumina 1.9+ format.",
          author: "bioinformaticist@bpipe.org"
  }
```
---
## The exec statement

#### Synopsis

    
    exec <shell command to execute>


#### Behavior

The *exec* statement runs the command specified as an argument using a bash shell in a managed fashion.  The command is specified using a string which may be in double quotes, single quotes or triple quotes.  [[Variables]] will be evaluated and expanded even when surrounded by single quotes.  Single quotes inside double quotes are passed through to the underlying shell and thus can be used to pass values that may contain spaces.  

The *exec* statement blocks until the specified command returns.  See the [[Async|async]] statement for an alternative that launches the process in the background and returns immediately.

Long commands that are passed to *exec* can be specified over multiple lines by using triple quotes.  Bpipe automatically removes newlines from commands so that you do not need to worry about it.

An *exec* statement will often be embedded in a [[Produce|produce]], [[Filter|filter]] or [[Transform|transform]] block to specify and manage the outputs of the command.

#### Logging

All commands executed via exec are automatically added to the command history for the run.

#### Failure

If the command fails or is interrupted (by Ctrl-C or kill), the pipeline will be automatically terminated.  Any outputs specified as generated by the stage running the exec command will be marked as dirty and moved to the local bpipe trash folder.  

#### Examples

**Example 1 - echo**

```groovy 

exec "echo 'Hello World'"
```

**Example 2 - Run Samtools**

```groovy 

exec """
   samtools mpileup -uf genome.fa $input | bcftools view -bvcg - > $output
"""
```

**Example 3 - Using Multiple Lines**
This example executes the same command as Example 2, but formats it over multiple lines.
```groovy 

exec """
    samtools mpileup 
        -u
        -f genome.fa 
        $input 
      | 
        bcftools view
         -b
         -v
         -c
         -g - > $output
"""
```
---
## Extension syntax for referencing input and output variables

### Introduction

When you use `$input` and `$output` you are asking for generic input and output file names from Bpipe, and it will give you names corresponding to defaults that it computes from the pipeline stage name. These, however, don't always end up with file extension that are very natural, and also may not be very robust because the type of file to be used is not specified. For example `$input` will refer to the *first* output from the previous pipeline stage - but what if that stage has multiple outputs and it changes their order? Your pipeline stage will get the wrong file. To help with this, you can add file extensions to your input and your output variables:
```groovy 

  align_reads = {
     exec "bowtie –S reference $input.fastq | samtools view -bSu - > $output.bam"
  }
```

Here we have specified that this pipeline stage expects an input with file extension ".fastq" and will output a file with file extension ".bam". The first will cause Bpipe search back through previous pipeline stages to find a file ending in .fastq to provide as input. The second will ensure that the output file for this stage ends with a ".bam" file extension.  Using these file extensions is optional, but it makes your pipeline more robust.

### Multiple Inputs

When a pipeline stage receives multiple inputs of various kinds, you can use the *$inputs* variable with extension syntax to filter out only those with the file extension you are interested in.

For example, to pass all the inputs ending with '.bam' to a 'merge_bam' command:
```groovy 

  merge_bams = {
     exec "merge_bams.sh $inputs.bam"
  }
```

### Limitations

At the moment extension syntax only works to filter a single file extension. Hence you can't ask for *$input.fastq.gz* - this will cause an error. For these cases you can consider using the [[From|from]] statement to match the right inputs:

```groovy 

  align_reads = {
     from("fastq.gz") {
         exec "bowtie –S reference $input.gz | samtools view -bSu - > $output.bam"
     }
  }
```

Here we can be sure we'll only match files ending in ".fastq.gz" in our bowtie command.

---
## The succeed statement

#### Synopsis

    
<pre>
    fail <message>
    fail [text {<text>} | html { <html> } | report(<template>)] to <notification channel name>
fail [{<text>} | html { <html> } | report(<template>)](text) to channel:<channel name>, 
                                                               subject:<subject>, 
                                                               file: <file to attach> 
</pre>
 
#### Availability

0.9.8.6_beta_2 +

#### Behavior

Causes the current branch of the pipeline to terminate explicitly with a failure status and a provided message.

In the most simple form, a short message is provided as a string. The longer forms allow a notification or report to be generated as a result of the success.

While using `fail` as a stand alone construct is possible, the primary use case is to embed it inside the otherwise clause of a [[Check|check]] command, which ensures that Bpipe remembers the status and output of the check performed.

*Note*: see the [[Send|send]] command for more information and examples about the variants of this command that send notifications and reports.

#### Examples

**Cause an Explicit Failure of the Pipeline**
```groovy 

   fail "Sample $branch.name has no variants - processing cannot continue"
```
---
## The File Statement

    
    file(< file path > )

#### Availability

Bpipe version 0.9.8.6

#### Behavior

A convenience function that creates a Java [File](http://docs.oracle.com/javase/6/docs/api/java/io/File.html) object for the given value. This is nearly functionally equivalent to simply writing '`new File(value)`', however it also converts the given path to a sane, canonicalised form by default, which the default constructor does not do, so that values such as `file(".").name` produce expected results. Bpipe does not check that the file exists or is a valid path.

#### Examples

Pass the full path of the current working directory to a command that requires to know it.
**
```groovy 

hello = {
    exec """
        mycommand -d ${file(".").absolutePath} -i $input.bam -o $output.bam
    """
}
run { hello }
```
---
## The Filter Statement


#### Synopsis

    
    filter(<filter name>) {
        < statements to filter inputs >
    
    }
#### Behavior

Filter is a convenient alias for [[Produce|produce]] where the name of the output or outputs is deduced from the name of the input(s) by keeping the same file extension but adding a new tag to the file name.   For example, if you have a command that removes comment lines from a CSV file *foo.csv*, you can easily declare a section of your script to produce output *foo.nocomments.csv* by declaring a filter with name "nocomments".  In general you will use filter where you are keeping the same format for the data but performing some operation on the data.

The output(s) that are automatically deduced by *filter* will inherit all the behavior implied by the [[Produce|produce]] statement.

#### Annotation

You can also declare a whole pipeline stage as a filter by adding the Filter annotation prior to the stage in the form `@Filter(<filter name>)`.

#### Examples

**Remove Comment Lines from CSV File**
```groovy 

filter("nocomments") {
  exec """
    grep -v '^#' $input > $output
  """
}
```
---
## The using instruction

#### Synopsis

    
    
      forward <output1> <output2>...
    
  

#### Behavior

A forward instruction overrides the default files that are used for inputs to the next pipeline stage with files that you specify explicitly. You can provide a hard coded file name, but ideally you will pass files that are based on the `input` or `output` implicit variables that Bpipe creates automatically so that your pipeline stage remains generic. 

Bpipe uses heuristics to select the correct output from a pipeline stage that would be passed forward by default to the next stage as an input (assuming the next stage doesn't specify any constraints about what kind of input it wants).  Sometimes however, the output from a pipeline stage is not usually wanted by following stages or Bpipe's logic selects the wrong output. In these cases it is useful to override the default with your own logic to specify which output should become the default.

#### Examples

**Return a BAM file as next input instead of the index file created by the stage**
```groovy 

  index_bam = {
    transform(input.bam+'.bai') {
       exec """samtools index $input.bam"""
    }
    forward input.bam
  }
```
---
## The from statement

#### Synopsis

    
    
      from(<pattern>,<pattern>,...|[<pattern1>, <pattern2>, ...]) {
         < statements to process files matching patterns >
      }
    
  
    
    
      from(<pattern>,<pattern>,...|[<pattern1>, <pattern2>, ...])  [filter|transform|produce](...) {
         < statements to process files matching patterns >
      }
    

#### Behavior

The *from* statement reshapes the inputs to be the most recent output file(s) matching the given pattern for the following block.  This is useful when a task needs an input that was produced earlier in the pipeline than the previous stage, or other similar cases where your inputs don't match the defaults that Bpipe assumes.

Often a *from* would be embedded inside a *produce*, *transform*, or *filter* block, but that is not required. In such a case, *from* can be joined directly to the same block by preceding the transform or filter directly with the 'from' statement.

The patterns accepted by *from* are glob-like expression using `**` to represent a wildcard. A pattern with no wild card is treated as a file extension, so for example "`csv`" is treated as "`**.csv`", **but will only match the first (most recent) CSV file**. By contrast, using `*.csv` directly will cause all CSV files from the last stage that output a CSV file to match the first parameter. This latter form is particularly useful for gathering all the files of the same type output by different parallel stages.

When provided as a list, *from* will accumulate multiple files with different extensions.  When multiple files match a single extension they are used sequentially each time that extension appears in the list given. 

#### Examples

**Use most recent CSV file to produce an XML file**
```groovy 

  create_excel = {
    transform("xml") {
      from("csv") {
        exec "csv2xml $input > $output"
      }
    }
  }
```

**Use 2 text and CSV files to produce an XML file**
```groovy 

  // Here we are assuming that some previous stage is supplying
  // two text files (.txt) and some stage (possibly the same, not necessarily)
  // is supplying a CSV file (.csv).
  create_excel = {
      from("txt","txt","csv") {
        exec "some_command $input1 $input2 $input3 > $output.xml" // input1 and input2 will be .txt, input3 will be .csv
      }
  }
```

**Match all CSV files from the last stage that produces a CSV file**
```groovy 

  from("*.csv") transform("xml") {
        exec "csv2xml $inputs.csv > $output"
  }
```
---
## The glob function

#### Synopsis

    
    
      glob(<pattern>)
    

#### Availability

0.9.8_beta_1 and higher

#### Behavior

Returns a list of all the files on the file system that match the specified wildcard pattern. The pattern uses the same format as shell wildcard syntax.

The *glob* function is useful if you want to match a set of files as inputs to a pipeline stage and you are not able to achieve it using the normal Bpipe input variables. For example, in situations where you want to match a complicated set of files, or files that are drawn from across different pipeline branches that don't feed their inputs to each other. A call to *glob* can be used inside a [[From|from]] statement to feed the matched files  into the input variables so that they can be referenced in [[Exec|exec]] and other statements in pipeline stages.

*Note*: in general, it is better to use `from` directly using a wild card pattern if you can. Such a form searches backwards through the outputs of previous stages rather than directly on the file system. This is much more robust than just scanning the file system for files matching the pattern.

#### Examples

**Run a command with all CSV files in the local directory as input**
```groovy 

  process_csv = {
    from(glob("**.csv")) {
        exec "convert_to_xml.py $inputs > $output"
    }
  }
```

In this example, the *$input* variable contains all files matching the pattern *`**`.csv* from the local directory, including the original inputs, and intermediate outputs of the pipeline.

---
## The grep statement

#### Synopsis

    
    
        grep(<regular expression>) {
          < statements that execute for each line matching expression >
        }
    

or

    
    
        grep(<regular expression>)
    

#### Behavior

The *grep* statement is an internal convenience function that processes the input file line by line for each line matching a given regular expression.  

In the first form, the body is executed for each line in the input file that matches and an implicit variable *line* is defined.  The body can execute regular commands using [[Exec|exec]] or it can use native Groovy / Java commands to process the data.  An implicit variable *out* is defined as an output stream that can write to the current output file, making it convenient to use *grep* to filter and process lines and write extracted data to the output file.

In the second form *grep* works very much like the command line grep with both the input and output file taken from the *input* and *output* variables.   In this case, all matching lines from the input are written to the output.

#### Examples

**Remove lines containing INDEL from a file**
```groovy 

grep(".*") {
  if(line.indexOf("INDEL") < 0)
    out << line
}
```

**Create output file containing only INDELS from input file**
```groovy 

grep("INDEL")
```
---
Welcome to the bpipe-migration wiki!
---
## The Load Statement

    
    load < file name >
#### Behavior

Imports the pipeline stages, variables and functions in the specified file into your pipeline script.

*Note*: currently the contents of files loaded using `load` are not imported directly into the file as if included literally. Instead they are scheduled to be imported when you construct a pipline. This means that when you execute a `run` or `segment` command, Bpipe loads them at that point to make them available within the scope of the run or segment statement. The result of this is that you cannot refer to them with global scope directly. See Example 2 to understand this more.

*Note*: As of Bpipe 0.9.8.5, a file loaded with `load` can itself contain `load` statements so that you can build multiple levels of dependencies. Use this feature with caution, however, as there is no checking for cyclic depenencies, and thus it is possible to put Bpipe into an infinite loop by having two files that load each other.

#### Examples

**1. Include a file "dependencies.groovy" explicitly into your pipeline**
```groovy 

load "dependencies.groovy"

run {
  hello + world // hello and world are defined in dependencies.groovy
}
```

**2. Refer to a Variable Defined in an External File**

`dependencies.groovy:`

```groovy 

INCLUDE_REALIGNMENT=true
```

```groovy 

load "dependencies.groovy"

// This will not work! The INCLUDE_REALIGNMENT variable is not defined yet
//if(INCLUDE_REALIGNMENT)
//  alignment = segment { align + realign }
//else
//  alignment = segment { align }

// This will work - INCLUDE_REALIGNMENT is defined inside segment {}
alignment = segment {
  if(INCLUDE_REALIGNMENT)
    align + realign
  else
    align
}

run {
  alignment
}
```
---
## The multi statement

#### Synopsis

    
    
      multi <command>,<command>...
    
      multiExec <Groovy list of commands>
    
    

#### Availability

0.9.8+ 

#### Behavior

The *multi* statement executes multiple commands in parallel and waits for them all to finish. If any of the commands fail the whole pipeline stage fails, and all the failures are reported.

Generally you will want to use Bpipe's built in [[ParallelTasks|parallelization]] features to run multiple commands in parallel. However sometimes that may not fit how you want to model your pipeline stages. The *multi* statement let's you perform small-scale parallelization inside your pipeline stages.

*Note*: if you wish to pass a computed list of commands to *multi*, use the form *multiExec* instead (see example below).

#### Examples

**Using comma delimited list of commands**

```groovy 

hello = {
  multi "echo mars > $output1.txt",
        "echo jupiter > $output2.txt",
        "echo earth > $output3.txt"
}
```

**Computing a list of commands**

```groovy 

hello = {  
        // Build a list of commands to execute in parallel
        def outputs = (1..3).collect { "out${it}.txt" }

        // Compute the commands we are going to execute
	int n = 0
        def commands =[$it > ${outputs[n++]("mars","jupiter","earth"].collect{"echo)}"} 

        // Tell Bpipe to produce the outputs from the commands
	produce(outputs) {
	    multiExec commands
	}
}

run { hello }
```
---
## The Output Function

    
    output(<file name>)
    

#### Behavior

The `output` function defines an explicit output for a command. It is similar to using a [[Produce]] clause but can be used inline with the definition of a command, using the form `${output(...)}`. In some situations it may be clearer to define outputs in line this way since the definition is collocated with its use. However in many cases it will be clearer to a reader if a [[Produce]] clause is used to show very clearly what outputs are created.

#### Examples

**Cause output to go to file "foo.txt"**
```groovy 

  hello = {
      exec """echo world > ${output("foo.txt")}"""
  }
```
---
## Prefix Convenience Function

### Prefix Convenience Function

Some tools don't ask you for an output file directly, but rather they ask you for a *prefix* for the output file, to which they append a suffix.  For example, the `samtools sort` command requires you to give it the name of the output without the ".bam" suffix. To make this easy to deal with, Bpipe offers a "magic" ".prefix" extension that lets you refer to an output, but pass to the actual command the output trimmed of its suffix.

### Example

```groovy 

  sort_reads = {
     filter("sorted") {
         exec "samtools sort -b test.bam $output.prefix"
     }
  }
```

Here we are referring to the output "something.sorted.bam" but only passing "something.sorted" to the actual command.

The "prefix" function actually works on any string (or text) value in Bpipe scripts. So you can even write it like this:
```groovy 

    BASE_FILENAME="test.bam".prefix
```

**Note:** when you use an expression like `$output.prefix`, it is important to understand that Bpipe considers this a reference to the file in $output, not a reference to a file named as the value that `$output.prefix` evaluates to. This means that if your command does not actually create the file with the name that `$output` evaluates to, you may get an error reported by Bpipe, because this is what it is expecting. For this reason, take care when using the `prefix` construct in conjunction with output variables and use them only when your command will actually create a file with the value of `$output` after being passed a truncated file name, and not simply as a convenience to make a string / text value with the value of `$output` truncated.  

---
## The Produce Statement

    
    
        produce(<outputs>|[output1,output2,...]) {
          < statements that produce outputs >
        }
    

#### Behavior

The *produce* statement declares a block of statements that will be executed *transactionally* to create a given set of outputs.  In this context, "transactional" means that all the statements either succeed together or fail together so that outputs are either fully created, or none are created (in reality, some outputs may be created but Bpipe will move such outputs to the [[Trash|trash folder]]).  Although you do not need to use *produce* to use Bpipe, using *produce* adds robustness and clarity to your Bpipe scripts by making explicit the outputs that are created from a particular pipeline stage. This causes the following behavior:

- If a statement in the enclosed block fails or is interrupted, the specified outputs will be "cleaned up", ie. moved to the [[Trash|trash folder]]
- The implicit variable 'output' will be defined as the specified value(s), allowing it to be used in the enclosing block
- The specified output will become the default input to the next stage in the pipeline
- If the specified output already exists and is newer than all the input files then the produce block will not be executed.

*Note*:  in most cases it will make more sense to use the convenience wrappers [[Filter|filter]] or [[Transform|transform]] rather than using *produce* directly as these statements will automatically determine an appropriate file name for the output based on the file name of the input.


A wildcard pattern can also be provided as input to `produce`.  In such a case, the `$output` variable is not assigned a value, but after the `produce` block executes, the file system is scanned for files matching the wild card pattern and any files found that were not present before running the command are treated as outputs of the block.

*Note*: as Bpipe assumes ALL matching new files are outputs from the `produce` block, using a wild card pattern inside a parallel block should be treated with caution, as multiple executing pathways may "see" each other's files.

#### Examples

**Produce a single output**
```groovy 

produce("foo.txt") {
  exec "echo 'Hello World' > $output"
}
```

**Produce multiple outputs**
```groovy 

produce("foo.txt", "bar.txt") {
  exec "echo Hello > $output1"
  exec "echo World > $output2"
}
```
---
## Executing Inline R Code


#### Synopsis

    
    R {"""
        < R code >
    
       """}
    
#### Behavior

Executes the embedded `R` code by passing it to the `Rscript` utility. Tokens preceded with `$` that match known Bpipe variables are evaluated as Bpipe variables before passing the script through. Other such tokens are passed through unmodified. This has the effect that you can reference your Bpipe variables such as `$input` and `$output` directly inside your R code.

*Note*: Any variable defined in the Bpipe / Groovy namespace will get evaluated, including pipeline stages, parameters passed to the script, and others. So it is important to understand that this can lead to unexpected evaluation of variables inside the R code. This feature is intended as a simple way to inline small R scripts, for example, to quickly create a plot of some results. Larger R programs should be executed by saving them as files and running them directly using Rscript.

*Note*: By default Bpipe uses the R executable (or actually, the Rscript executable) that it finds in your path. If you want to set a custom R executable, you can do so by adding a bpipe.config file with an entry such as the following:
```groovy 

R {
    executable = "/usr/local/R/3.1.1/bin/Rscript"
}
```

#### Examples

**Plot the values from a tab separated file**
```groovy 

R {"""
  values = read.table('$input.tsv');
  png('$output.png')
  plot(values)
  dev.off()
"""}
```
---
## The Segment Statement

    
    segment_name = segment {
        < pipeline stage > + < pipeline stage > + ...
    
    }
#### Behavior

Defines a reusable segment of a pipeline that can be made up of multiple pipeline stages.  The segment then behaves just like a normal pipeline stage itself.   This statement is useful when you have a commonly reoccurring sequence of pipeline stages that you wish to use multiple times in your pipelines, or even multiple times in a single pipeline.

#### Examples

**Define a simple Pipeline Segment**
```groovy 

hello = {
   exec "echo hello"
}

world = {
   exec "echo world"
}

hello_world = segment {
  hello + world
}

Bpipe.run {
  hello_world
}
```
---
## The send statement

#### Synopsis

    
    
    send [text {<text>} | html { <html> } | report(<template>)] to <notification channel name><pre>
send [{<text>} | html { <html> } | report(<template>)](text) to channel:<channel name>, 
                                                               subject:<subject>, 
                                                               file: <file to attach> 
</pre>
 
#### Availability

0.9.8.6_beta_2 +

#### Behavior

Sends the specified content through the specified communication channel.

The first form sends the content using the defaults that are inferred from the configuration of the channel or derived directly from the content itself. The second form allows specification of lower level details of how the content should be sent and the message.

The purpose of `send` is to enable explicitly sending content such as small reports and status messages as part of the flow of your pipeline. The `send` command can occur inside pipeline stages that perform other processing, or stand alone as part of a dedicated pipeline stage.

The content can be specified in three different ways. The first option simply specifies a literal text string that is used as the message directly. Note that the text string *must* appear within curly braces. It will be lazily evaluated just prior to sending. The `html` option allows creation of HTML content programmatically. The body of the closure (that is, inside the curly braces) is passed a [Groovy MarkupBuilder](http://groovy.codehaus.org/Creating+XML+using+Groovy's+MarkupBuilder). This can be used to create HTML that forms the body of an HTML email. 

The final form allows specification of a template. The template should end with an extension appropriate to the content type (for example, to send an HTML email, make sure the template ends with ".html").  The template file is processed as a [Groovy Template](http://groovy.codehaus.org/Groovy+Templates) which allows references to variables using the normal `$` syntax, `${variable}` form as well as complete Groovy programmatic logic within `<% %>`  and `<%= %>` blocks.

*Note*: a common scenario is to terminate a branch of a pipeline due to a failure or exceptional situation and to send a status message about that. To make this convenient, the [[Succeed|succeed]] and [[Fail|fail]] commands can accept the same syntax as `send`, but also have the effect of terminating the execution of the current pipeline branch with a corresponding status message.

#### Examples

**Send a message via Google Talk**
```groovy 

    send text {"Hello there"} to gtalk
```

**Send a message via Gmail, including a subject line**
```groovy 

    send text {"Hello there, this is the message body"} to channel: gmail, subject: "This is an email from Bpipe"
```

**Send an HTML email to Gmail**
```groovy 

    send html {
        body { 
           h1("This Email is from Bpipe")
           table { 
               tr { th("Inputs") }
               inputs.each { i -> tr { td(i) }  }
           }
    } to gmail
```

**Send a message based on a template using Gmail, and attach the first output as a file**
```groovy 

    send report("report-template.html") to channel: gmail, file: output1.txt
```
---
## The succeed statement

#### Synopsis

    
    
    succeed <message>
    succeed [text {<text>} | html { <html> } | report(<template>)] to <notification channel name><pre>
succeed [{<text>} | html { <html> } | report(<template>)](text) to channel:<channel name>, 
                                                               subject:<subject>, 
                                                               file: <file to attach> 
</pre>
 
#### Availability

0.9.8.6_beta_2 +

#### Behavior

Causes the current branch of the pipeline (or the whole pipeline, if executed in the root branch) to terminate with a successful status, but without producing any outputs.

In the most simple form, a short message is provided as a string. The longer forms allow a notification or report to be generated as a result of the success.

The `succeed` command allows you to have a branch of your pipeline terminate without continuing or feeding outputs into any following stages. This is mostly useful in situations where you have many parallel stages running. In a normal case, Bpipe expects every parallel branch to produce an output, and will fail the entire pipeline if the expected outputs are not generated. Sometimes however, no outputs are legitimately produced and in that case you just want to stop processing in those branches that give no outputs while allowing others to continue without aborting the whole pipeline.

While using `succeed` as a stand alone construct is possible, the primary use case is to embed it inside the otherwise clause of a [[Check|check]] command, which ensures that Bpipe remembers the status and output of the check performed.

*Note*: see the [[Send|send]] command for more information and examples about the variants of this command that send notifications and reports.

#### Examples

**Terminate Branch Successfully**
```groovy 

   succeed "Sample $branch.name has no variants"
```
---
## The Transform Statement

    
    transform(<transform name>) {
        < statements to transform inputs >
    
    }

    
    transform(<input file pattern>) to(replacement pattern)  {
        < statements to transform inputs >
    
    }
#### Behavior

Transform is a convenient alias for [[Produce|produce]] where the name of the
output or outputs is deduced from the name of the input(s) by modifying the
file extension.   For example, if you have a command that converts 
a CSV file called *foo*.csv to an XML file, you can easily declare a section of your script
to output *foo*.xml using a transform with the name 'xml'.

The output(s) that are automatically deduced by *transform* will inherit all the behavior implied by the [[Produce|produce]] statement.

Since version 0.9.8.4, transform has offered an extended form that allows you to do more than just replace the file extension. This form uses two parts, taking the form: 
  
  `transform(<input file pattern>) to(<output file pattern>`) { ... }

The input and output patterns are assumed to match to the end of the file name, but can include a regular expression pattern for matching the input files.

*Note*: input file patterns that contain no regular expression characters, or that end in "." followed by plain characters are treated as file extensions. ie: ".xml" is treated as ".xml", not "any character followed by xml".

#### Annotation

You can also declare a whole pipeline stage as a transform by adding the Transform annotation prior to the stage in the form `@Transform(<filter name>)`. This form is a bit less flexible, but more concise when you don't need the flexibility.

#### Examples

**Remove Comment Lines from CSV File**
```groovy 

transform("xml") {
  exec """
    csv2xml $input > $output
  """
}
```

**Run FastQC on a Gzipped FASTQ file**
Fastqc produces output files following an unusual convention for naming. To match this convention, we can use the extended form of transform:
```groovy 

fastqc = {
    transform('.fastq.gz') to('_fastqc.zip') {
        exec "fastqc -o . --noextract $inputs"
    }
    forward input
}
```

Note also that since the output zip files from FastQC are usually not used downstream, we forward the input files rather than the default of letting the output files be forwarded to the next stage.

---
## The using instruction

#### Synopsis

    
    
      <pipeline stage>.using(<variable>:<value>, <variable2>:value...)
    
  

#### Behavior

A *using* instruction causes variables to be defined inside a pipeline stage as part of pipeline construction.  It is only valid inside a Bpipe *run* or [[Segments|segment]] clause.  The *using* instruction is useful to pass parameters or configuration attributes to pipeline stages to allow them to work differently for different pipelines.

#### Examples

**Sleep for a configurable amount of time and say hello to a specified user**
```groovy 

  hello = {
    exec "sleep $time"
    exec """echo "hello $name" """
  }

  run {
    hello.using(time: 10, name: "world") + hello.using(time:5, name:"mars")
  }
```
---
## Variables in Bpipe

Bpipe supports variables inside Bpipe scripts.  In Bpipe there are two kinds of variables: 

- *implicit* variables  - variables defined by Bpipe for you
- *explicit* variables  - variables you define yourself

#### Implicit Variables

Implicit variables are special variables that are made available to your Bpipe pipeline stages automatically.   The two important implicit variables are:

- input - defines the name(s) of files that are inputs to your pipeline stage
- output - defines the name(s) of files that are outputs from your pipeline stage
- environment variables - variables inherited from your shell environment (note: this feature is not implemented yet - coming soon!).
- branch - when a pipeline is running parallel branches, the name of the current branch is available in a *$branch* variable.

The input and output variables are how Bpipe automatically connects tasks together to make a pipeline.  The default input to a stage is the output from the previous stage. In general you should always try to use these variables instead of hard coding file names into your commands.   Using these variables ensures that your tasks are reusable and can be joined together to form flexible pipelines.  

#### =Extension Syntax for Input and Output Variables=

Bpipe provides a special syntax for easily referencing inputs and outpus with specific file extensions. See [[ExtensionSyntax]] for more information.

**Multiple Inputs**

Different tasks have different numbers of inputs and outputs, so what happens when a stage with multiple outputs is joined to a stage with only a single input?  Bpipe's goal is to try and make things work no matter what stages you join together.  To do this:

- The variable *input1* is always a single file name that a task can read inputs from.  If there were multiple outputs from the previous stage, the *input1* variable is the first of those outputs.  This output is treated as the "primary" or "default" output.  The variable `$input` also evaluates, by default, to the value of *input1*.
- If a task wants to accept more than one input then it can reference each input using the variables *input1*, *input2*, etc.
- If a task has an unknown number of inputs it may reference the variable *input* as a *list* and use array like indices to access the elements, for example `_input[is the same as `input1`, `input[1](0]_`)` corresponds to `input2`, etc.   You can reference the size of the inputs using `input.size()` and iterate over inputs using a syntax like
```groovy 

  for(i in input) {
    exec "... some command that reads $i ..."
  }
```

- When a stage has multiple outputs, the first such output is treated as the "primary" output and appears to the next stage as the "input" variable.  If the following stage only accepts a single input then its input will be the primary output of the previous stage.

#### Explicit Variables

Explicit variables are ones you define yourself.  These variables are created inline inside your Bpipe scripts using Java-like (or Groovy) syntax.  They can be defined inside your tasks or outside of your tasks to share them between tasks.  For example, here two variables are defined and shared between two tasks:
```groovy 

  NUMTHREADS=8
  REFERENCE_GENOME="/data/hg19/hg19.fa"

  align_reads = {
    exec "bwa aln -t $NUMTHREADS $REFERENCE_GENOME $input"
  }
  
  call_variants = {
    exec "samtools mpileup -uf $REFERENCE_GENOME $input > $output"
  }
```

#### Variable Evaluation

Most times the way you will use variables is by referencing them inside shell commands that you are running using the *exec* statement.  Such statements define the command using single quotes, double quotes or triple quotes.  Each kind of quotes handles variable expansion slightly differently:

**Double quotes** cause variables to be expanded before they are passed to the shell.  So if input is "myfile.txt" the statement: 
```groovy 

  exec "echo $input"
```

will reach the shell as exactly:
```groovy 

  echo myfile.txt
```

Since the shell sees no quotes around "myfile.txt" this will fail if your file name contains spaces or other characters treated specially by the shell.  To handle that, you could embed single quotes around the file name:
```groovy 

  exec "echo '$input'"
```

**Single quotes** cause variables to be passed through to the shell without expansion.  Thus 
```groovy 

  exec 'echo $input'
```

will reach the shell as exactly:
```groovy 

  echo $input
```

**Triple quotes** are useful because they accept embedded newlines.  This allows you to format long commands across multiple lines without laborious escaping of newlines.   Triple quotes escape variables in the same way as single quotes, but they allow you to embed quotes in your commands which are passed through to the shell.  Hence another way to solve the problem of spaces above would be to write the statement as:
```groovy 

  exec """
    echo "$input"
  """
```

See the [[Exec|exec]] statement for a longer example of using triple quotes.

#### Referencing Variables Directly

Inside a task the variables can be referenced using Java-like (actually Groovy) syntax.  In this example Java code is used to check if the input already exists:
```groovy 

  mytask = {
      // Groovy / Java code!
      if(new File(input).exists()) {
        println("File $input already exists!")
      }
  }
```

You won't normally need to use this kind of syntax in your Bpipe scripts, but it is available if you need it to handle complicated or advanced scenarios.

#### Differences from Bash Variable Syntax

Bpipe's variable syntax is mostly compatible with the same syntax in languages like Bash and Perl. This is very convenient because it means that you can copy and paste commands directly from your command line into your Bpipe scripts, even if they use environment variables.

However there are some small differences between Bpipe variable syntax and Bash variable syntax:

- Bpipe always expects a `$` sign to be followed by a variable name.  Thus operations in Bash that use `$` for other things need to have the `$` escaped so that Bpipe does not interpret it.  For example:

```groovy 

  exec "for i in $(ls **.bam); do samtools index $i; done"
```

In this case the $ followed by the open bracket is illegal because Bpipe will try to interpret it as a variable.  We can fix this with a backslash:
```groovy 

  exec "for i in \$(ls **.bam); do samtools index $i; done"
```

- Bpipe treats a `.` as a special character for querying a property, while Bash merely delimits a variable name with it. Hence if you wish to write `$foo.txt` to have a file with '.txt' appended to the value of variable foo, you need to use curly braces: `${foo}.txt`.

---
## The Preserve Statement

    
    preserve(<substring>) {
        < statements for which outputs should be preserved > 
    }

    
    @preserve
    stage_name = { 
        <statements> 
    }

#### Behavior

Causes the outputs generated by statements inside the block to be marked as preserved, and thus will not be automatically cleaned up by a `bpipe cleanup` command. 

#### Examples

**Ensure the VCF file of a Variant Calling Operation is not Deleted by Cleanup**

```groovy 

preserve("*.vcf") {
        exec """ 
                java -Xmx8g -jar GenomeAnalysisTK.jar -T UnifiedGenotyper 
                   -R $REF 
                   -I $input.bam 
                   -nt 4
                   --dbsnp $DBSNP 
                   -dcov 1600 
                   -l INFO 
                   -A AlleleBalance -A Coverage -A FisherStrand 
                   -glm BOTH
                   -metrics $output.metrics
                   -o $output.vcf
            """
}
```
---
## The Requires Statement

    
    requires < variable name > : Message to display if value not provided

#### Availability

Bpipe version 0.9.8.6

#### Behavior

Specify a parameter or variable that must be provided for this pipeline stage to run. The variable can be provided with a [[Using]] construct, by passing a value on the command line (--param option), or simply by defining the variable directly in the pipeline script. If the variable is not defined by one of these mechanisms, Bpipe prints out an error message to the user, including the message defined by the `requires` statement.

#### Examples

**This Pipeline will Print an Error because WOLRD is not defined anywhere**
```groovy 

hello = {
    requires WORLD : "Please specify which planet to say hello to"

    doc "Run hello with to $WORLD"

    exec """
        echo hello $WORLD
    """
}
run { hello }
```

**Say hello to mars**
Note that the script below is identical to the example above, except for the "run" portion.
```groovy 

hello = {
    requires WORLD : "Please specify which planet to say hello to"

    doc "Run hello with to $WORLD"

    exec """
        echo hello $WORLD
    """
}
run { hello.using(WORLD: "mars") }
```

**Say hello to mars with a directly defined variable **
Note that the script below is identical to the example above, except for the "run" portion.
```groovy 

hello = {
    requires WORLD : "Please specify which planet to say hello to"

    doc "Run hello with to $WORLD"

    exec """
        echo hello $WORLD
    """
}

WORLD="mars"

run { hello }
```
---
## The uses statement

#### Synopsis

    
    
      uses(<resource>,<resource>...) {
         < statements that use resources >
      }
    

#### Availability

0.9.8_beta_3 and higher

#### Behavior

The *uses* statement declares that the enclosed block will use the resources declared in brackets. The resources are specified in the form *type*:integer where predefined types are "threads", "MB" or "GB". The latter two refer to memory. Custom resource types can be used by simply specifying arbitrary names for the resources.

Once resources have been declared, any [[Exec|exec]] statement that appears in the body is assumed to require the resources and will block if the more of the resource are concurrently in use than allowed by the maximums specified on the command line, or in configuration (bpipe.config).

The purpose of this statement is to help you control concurrency to achieve better utilization and prevent over-utilization of server resources, particularly when different parts of your pipeline have different resource needs. Some stages may be able to run many instances in parallel without overloading your server, while others may only be able to run a small number in parallel without overloading the system. You can solve these problems to get consistently high utilization throughout your pipeline by adding *uses* blocks around key parts of your pipeline.

#### Examples

**Run bwa with 4 threads, ensuring that no more than 16GB, 12 threads and 3 temporary files are in use at any one time**
```groovy 

  run_bwa = {
    uses(threads:4,GB:4,tempfiles:2) {
        exec "bwa aln -t 4 ref.fa $input.fq"
    }
  }
```

Note that "tempfiles" is a custom resource (bwa does not really create temporary files, we just do this for the sake of example).

The pipeline would be executed with:

```groovy 

    bpipe run pipeline.groovy -n 12 -m 16384 -l tempfiles=3 test.fq 
```
---
## The Var Statement

    
    var < variable name > : <default value>

#### Availability

Bpipe version 0.9.8.1

#### Behavior

Define a variable with a default value, that can be overridden with a [[Using]] construct, or by passing a value on the command line (--param option).

#### Examples

**Say hello to earth**
```groovy 

hello = {
    var WORLD : "world"

    doc "Run hello with to $WORLD"

    exec """
        echo hello $WORLD
    """
}
run { hello }
```

**Say hello to mars**
Note that the script below is identical to the example above, except for the "run" portion.
```groovy 

hello = {
    var WORLD : "world"

    doc "Run hello with to $WORLD"

    exec """
        echo hello $WORLD
    """
}
run { hello.using(WORLD: "mars") }
```
---
