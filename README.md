# :sparkles:Making Data, the DataMade Way:sparkles:

## Contents
- [Intro](#intro)
- [DataMade's Data Making Principles](#datamades-data-making-principles)
- [Make & Makefiles](#make--makefiles)
  - [Why Use Make/Makefiles?](#why-use-makemakefiles)
  - [Makefile 101](#makefile-101)
  - [Makefile 201 - Some Fancy Things Built Into Make](#makefile-201---some-fancy-things-built-into-make)
- [DataMade ETL Styleguide](#datamade-etl-styleguide)
  - [Makefile Best Practices](#makefile-best-practices)
  - [Variables](#variables)
  - [Processors](#processors)
  - [Standard Toolkit](#standard-toolkit)
  - [ETL Workflow Directory Structure](#etl-workflow-directory-structure)

## Intro

This is documentation for the DataMade ETL workflow.

ETL refers to the general process of:

1. taking raw **source data** (Extract)
2. doing some stuff to get the data in shape, possibly involving intermediate **derived files** (Transform)
3. & ultimately ending up with **final output** in a usable form (for Loading into something that consumes the data - be it an app, a system, a visualization, etc.)

For enthralling insights on how to get from source data to final output, all while minimizing future headaches - read on!

## DataMade's Data Making Principles

- Treat inputs as immutable - don't modify source data directly
- Be able to deterministically produce the final data with one command 
- Write as little custom code as possible 
- Use [standard tools](#standard-toolkit) whenever possible
- Source data should be under version control

## Make & Makefiles

To achieve a reproducible data workflow, we use GNU make

### Why Use Make/Makefiles?
A simple way of thinking about a data processing workflow is as a series of steps. However, instead of thinking *forward*, in terms of an order of steps from step 1 to step N, you can also also think *backwards* - in terms of the outputs that you want and the files that those outputs are derived from. Thinking backwards is a more powerful way of expressing a data workflow, since dependencies aren't always linear.

```make``` is a build tool that generates file *targets*, each of which can depend upon the existence of other files (*dependencies*). Targets, dependencies, and instructions specifying to build them are defined in a *makefile*. The nice thing about makefiles is that once you specify a dependency graph, ```make``` will do the work of figuring out the individual steps required to build an output, based on your rules and the files you already have.

**```make``` is a particularly nifty tool for data processing because**:
- ```make``` allows you to create all final data with a single command, since ```make``` rules can be chained. writing a ```makefile``` is ultimately an exercise in making your existing data processing steps explicit, to ultimately avoid manual, undocumented steps
- ```make``` is smart about only building what's necessary, because it's aware of when a file was last modified - ```make``` will not rebuild existing files if their dependencies haven't changed.
- ```make``` give you parallel processing for nearly free


### Makefile 101
When you run a ```make``` command, ```make``` will look for instructions in a file called ```Makefile``` in the current directory. The building block of a makefile is a "rule". Each "rule" specifies (1) a *target*, (2) the target's *dependencies*, and the target's *recipe* (i.e. the commands for creating the target).

**The general structure of a single make "rule":**
```
target: dependencies
[tab] recipe
```
**Targets** - the target is what you want to generate. ```make``` expects all targets to be files, with the exception of [phony target](https://github.com/datamade/data-making-guidelines#phony-targets). a file target can be an output filename, an output file pattern, or a [variable](https://github.com/datamade/data-making-guidelines#variables).  
**Dependencies** - dependencies are everything that needs to exist in order to make the target. ```make``` expects all dependencies to be files. dependencies can be filenames, filename patterns, or [variables](https://github.com/datamade/data-making-guidelines#variables). dependencies are optional.   
**Recipes** - recipes are commands for generating the target file. any command you can run on the terminal is fair game  for recipes - bash commands, invoking a script, etc.  

### Makefile 201 - Some Fancy Things Built Into Make

The following is not complete documentation of ```make``` functionality - just some stuff we use most often.

#### Phony Targets

By default, ```make``` assumes that targets are files. However, sometimes it is useful to run commands that do not represent physical files - for example, making all targets or cleaning your directory. To define phony targets, you must explicitly tell ```make``` that they are not associated with files, like so:
```
.PHONY: all clean
```
The most common examples of phony targets that we use are ```all``` (make all targets defined in the makefile) and ```clean``` (clean all derived files).

Example rules for these common commands:
```
all: $(GENERATED_FILES)

clean:
	rm -Rf finished/*
```
*Note: for the ```$(GENERATED_FILES)``` dependency, ```GENERATED_FILES``` should be a variable defined to include all final output targets in a makefile*

#### Automatic Variables
GNU make comes with some [automatic variables](http://www.gnu.org/software/make/manual/html_node/Automatic-Variables.html#Automatic-Variables) that you can use *in your recipe* to refer to specific targets/dependencies.

The most common automatic variables we use:

| variable | what it refers to |
|---|---|
| ```$@``` | the filename of the target |
| ```$^``` | the filenames of all dependencies |
| ```$?``` | the filenames of all dependencies that are newer than the target |
| ```$<``` | the filenames of the first dependency |

#### Pattern Rules (Implicit Rules)

In cases where you don't want to state targets explicitly, you can write an [implicit rule](https://www.gnu.org/software/make/manual/html_node/Pattern-Rules.html) by including ```%``` in the target and dependencies - ```%``` will match any nonempty substring, and the match is called the *stem*.

#### Functions for Filenames

There are some convenient [functions](https://www.gnu.org/software/make/manual/html_node/File-Name-Functions.html) for working with a filename or multiple filenames.

Some useful filename functions:

| filename function | what it does |
|---|---|
| ```$(dir [filepaths])``` | returns only the directory path |
| ```$(notdir [filepaths])``` | returns only the file name |

For example, ```$(dir finished/file1.csv finished/file2.csv)``` = ```'finished/ finished/'``` & ```$(notdir finished/file1.csv finished/file2.csv)``` = ```'file1.csv file2.csv'```

## DataMade ETL Styleguide

### Makefile Best Practices

Some loose notes on best practices:
- Some transformations, especially those chaining unix tools, are obscure. Consider printing the purpose of the transformation ```@echo "Downcasing the header of this csv"```
- Always echo commands.
- To limit verbosity, use arg flags. Avoid piping stderr to dev/null
- List recipes in rough order of processing steps
- Have 'all' and 'clean' targets
- Prefer implicit patterns over explicit recipes. Encourages DRY and files created by implicit patterns will automatically be cleaned up. 
- Makefile directives go at the top file, followed by variables go at the top of the file, followed by 'all' and 'clean' targets
- Use these Makefile directives
```make
MAKEFLAGS += --warn-undefined-variables
SHELL := bash
.SHELLFLAGS := -eu -o pipefail
.DEFAULT_GOAL := all
.DELETE_ON_ERROR:
.SUFFIXES:
```


### Variables
Variables are names defined in a makefile to refer to files, directories, targets, or just about anything that you can represent with text.

**A few common variables we define:**

| variable | description |
|---|---|
| ```GENERATED_FILES``` | a list of all the final output targets that the makefile can build. this is used as a shorthand way of calling everything in the ```all``` [phony target](https://github.com/datamade/data-making-guidelines#phony-targets) |
| ```DATA_DIRS``` | if there is a master makefile and sub-makefiles, this includes all of the sub-directories. it is useful for the ```all``` [phony target](https://github.com/datamade/data-making-guidelines#phony-targets) in the master makefile |
| ```DIR``` | points to the directory that contains the master makefile |
| ```PYTHON_BIN``` | if a Python Virtual Environment needs to be created, this points to the ```bin/``` directory inside that virtual environment |
| ```PROCESSOR_DIR``` | if the workflow requires [processors](https://github.com/datamade/data-making-guidelines#processors), this points at the directory containing the processors |

If you have a master makefile and multiple sub-makefiles, you should define ```GENERATED_FILES``` in each sub-makefile, and the other variables above in the master makefile.

### Processors
When processing a target requires more than can be accomplished with our [standard toolkit](https://github.com/datamade/data-making-guidelines#standard-toolkit), a processor (i.e. a script for a single operation) can be written.

For the sake of easier reuse, each processor should be modular, only handling one operation on a file. Each processor should be configured to accept input on ```STDIN``` and write output to ```STDOUT```, so that it's easy to chain processors and operations.

All processors should live in a ```processors/``` directory in the root of the repository. To make processors available to all makefiles, define the path to ```processors/``` in the ```PROCESSORS``` [variable](https://github.com/datamade/data-making-guidelines#variables).

Some examples of single-purpose processors:
- [excel date column -> ISO formatted date column](https://github.com/datamade/gary-counts-data/blob/master/data/processors/convert_excel_time.py)
- ['NA' or 'N/A' -> None](https://github.com/datamade/gary-counts-data/blob/master/data/processors/make_real_nulls.py)
- [delete empty rows from a csv](https://github.com/datamade/gary-counts-data/blob/master/data/processors/delete_empty_rows.py)
- [strip whitespace in a csv](https://github.com/datamade/gary-counts-data/blob/master/data/processors/strip_whitespace.py)

## Standard Toolkit

- For fetching content on the web, use wget. Use `--no-use-server-timestamps` arg for wget.
- For manipulating geo files, use GDAL/OGR 
- CSVKit for spreadsheets, or things that can be made into spreadsheets. In particular
  -  [```in2csv```](https://csvkit.readthedocs.org/en/0.9.1/scripts/in2csv.html)    
  -  [```csvcut```](https://csvkit.readthedocs.org/en/0.9.1/scripts/csvcut.html)
  -  [```csvjoin```](https://csvkit.readthedocs.org/en/0.9.1/scripts/csvjoin.html) 
- For simple sql-like queries use csvkit
- For more complicated queries use postgres
- For geospatial queries use postgis
- For text manipulation use sed, unless it's **much** easier to do it with awk
- CSVKit for spreadsheets, or things that can be made into spreadsheets
- unzip, gzip, and tar for uncompressed files. If you are compressing files, and have an option, use tar zcvf
- For custom transform code, use Python

### ETL Workflow Directory Structure

In the case that a project has multiple separate data components, you can define a master makefile at the root of the repository, along with sub-directories that each have a sub-makefile at the root. When using this type of nested structure, all data processing/transformation should be defined in the sub-makefiles - the master makefile should only handle setting up the environment, defining variables/targets used by multiple sub-makefiles, & calling sub makefiles.

```
|-- Makefile  # the master makefile
|-- README.md
|-- data/
|   |-- <sub-directory for a data processing component>
|   |   |-- Makefile   # a sub-makefile
|   |   |-- README.md  # documents the data source & how data gets processed
|   |   |-- finished/  # a directory for finished files
|   |-- <sub-directory for another data processing component>
|   |   |-- Makefile   # a sub-makefile
|   |   `-- < ... etc ... >
|-- processors
|   |-- <processor_name>.py
|   `-- <another_processor_name>.sh
`-- requirements.txt   # lists install requirements for the pipeline
```


## Example Repositories
- [Gary Counts](https://github.com/datamade/gary-counts-data)
- [Trees](https://github.com/fgregg/trees)

## Related Links
- [Makefile Style Guide by Clark Grubb](http://clarkgrubb.com/makefile-style-guide#data-workflows)
- [Why Use Make by Mike Bostock](http://bost.ocks.org/mike/make/)
