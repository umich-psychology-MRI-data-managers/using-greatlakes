# Using job arrays on Great Lakes

Job arrays are a way of creating many jobs from a single job script.

An array will have some number of job elements, say five of them.
They can be numbered most simply 1, 2, 3, 4, 5.  In your job script
you would indicate that with a preamble entry that says

```bash
#SBATCH --array=1-5
```

You could also list five separate, integer elements separated by
commas with no spaces, e.g.,

```bash
#SBATCH --array=21,36,38,43,59
```

The numbers in the `--array` specification are called array task IDs,
and they become part of the job name when the array is expanded
into jobs.

If I submit the first job, with `1-5`, and the Slurm JobID is
`1234`, then the jobs that will run are

```
1234_1
1234_2
1234_3
1234_4
1234_5
```

and in the second case, they would be

```
1234_21
1234_36
1234_38
1234_43
1234_59
```

Each of the individual jobs are indpendent of the others and
will run as if you wrote a separate job script for each and
submitted it yourself.

However, each array job will have an environment variable called
`SLURM_ARRAY_TASK_ID` that can be used in the job.  One simple
way to use it would be if you had data files that contained a
number in them and you wanted to do something with each of those
files.  In your job script, you could use

```
my_program /my/data/subject_${SLURM_ARRAY_TASK_ID}.dat
```

Each job would replace `${SLURM_ARRAY_TASK_ID}` with the value
of the current job's array task ID.

It's more often the case that you want to process a list of
subjects.  In that case, you create a file that contains a subject
list, e.g., `subs.txt` containing, say,

```
sub10123
sub10124
sub10131
sub10142
sub10259
```

Using the simple case where you have five entries, and you use
`--array=1-5`, you need each job to count down from the top of
the file however many lines the array task ID says, then set
the value of `$subj` to that line.

We use two commands from Linux to do this.  First we use

```bash
head -${SLURM_ARRAY_TASK_ID} subs.txt
```
to print out the first `${SLURM_ARRAY_TASK_ID}` lines from the
file.  So, if it were `2`, then it would print

```
sub10123
sub10124
```

and if it were `5` (or larger), it would print all five lines.

If it is `2`, we only want the second line; if is `3`, only the
third.  So, we use a Linux pipe (a way to send the output of
one command as input to another), which is indicated by the
'pipe' character(`|`), also sometimes called the vertical bar.
The command to which we want to send the output of `head` is
called `tail`, and it will print the _last_ lines; in this
case, if we have `head` print three lines, and we send those
three lines to `tail` and ask it to print just the last line,
that will be the third line.

```bash
head -${SLURM_ARRAY_TASK_ID} subs.txt | tail -1
```

Finally, if we enclose that in a subshell, we can assign the
output to a variable, as in this example where we use
the variable `subj`.

```bash
subj=$(head -${SLURM_ARRAY_TASK_ID} subs.txt | tail -1)
```

We can then proceed to do something useful to that subject, like

```
recon-all $subj
```

In addition to processing all the lines in the file, we could
select lines using the Linux `grep` command.  `grep` searches
of occurences of its first argument in the file(s) in the
second argument.

With just a single file, it will print all the lines that have
the first argument in them.  So, if you used `grep 102 subs.txt`,
it would only find `sub10259`. If, instead you used `grep 101
subs.txt`, it would find the other four lines and not `sub10259`.

For example, suppose only subjects 10124 and 10259 needed to be
run. We could use `--array=10124,10259` and then use

```
subj=$(grep ${SLURM_ARRAY_TASK_ID} subs.txt)
```

to extract just those two subjects from `subs.txt`, and those
would then be processed.
