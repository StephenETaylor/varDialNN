Intend to work on the varDial shared task in this directory.
described at: http://ttg.uni-saarland.de/vardial2016/dsl2016.html


My plan is to write an lstm rnn which will classify Arabic transcripts.
I think it will be a variant of the Karpathy Torch char-rnn,
http://karpathy.github.io/2015/05/21/rnn-effectiveness/

   1)The loss function is closely related to the cross entropy between the
   training text and the test text so one could use the code unmodified,

   2)but I plan to rewrite the output of the model as a classifier,
   perhaps with 5 soft-max outputs or conceivably as 5 separate nn's,
   each classifying yes/no. (two soft-max outputs).

8/Aug/2016.  I wrote and sent off a de-Buckwalter-er today.  I think I
should split off some test data, some validation data.

wrote some python scripts, all in varDialTrainingData:
xliterate.py -- mentioned above, turns training file into 5 dialect files
split-task.py -- splits training file into training, validation, testing files
testform.py -- transforms a test file to drop out the correct answers.


makefile sort of documents usage of scripts.

To train with this code:
th train.lua -seq_length 520 -data_dir varDialTrainingData -gpuid -1

where  
  th invokes the Torch package (see http://torch.ch/)
  train.lua is the top-level code, but it also calls code in model, util, model-util, misc directories
  -seq_length 520 is an effort to include at least the first 520 characters of each training item.  much smaller sequence lengths are possible, but this 
length discards little of the training data.  The median sequence length in
the training data is about 300 -- I think 520 is beyond the 90th percentile.
  The varDialTrainingData directory includes the "training" file, which 
is used for training data.
  -gpuid -1 indicates that my laptop does not have an nVidia graphics card,
and I am using the CPU for training.

The code defaults to producing a checkpoint after every 1000 batches.
It's supposed to run for 50 epochs, but there may not be time enough
before August 31.  The validation loss function does seem to be decreasing
slowly.

Wrote test.lua which can specify a model; it accepts test lines from
standard input and writes standard output.  

varDialTrainingData/statistics.lua accepts the name of a test output file on
the command line [e.g. th statistics.lua ../a2] and compares against a 
(reserved from the training data) file "training" -- which contains the 
truth as offered by the workshop.

As of 18 August 2016, the train.lua script bombs out after a little more than
one epoch of training, announcing that the loss function is blowing up.
I'm going to play with this, but don't know what to do next.

Decided to modify the training data.  The problem seems to be a wide
variation in the loss functions for sequences longer than 1000 characters.
Since we're not training with the earlier part of such sequences, it
makes sense to eliminate them somehow.  I created two variant training files,
training.s and training.t.   Training.s has all sequences greater than 490
characters split at the next space.  Training.t truncates all sequences
greater than 490 characters at the next space. Both approaches eliminate
the long sequences; the .s file may include some code-switching, as longer
sequences provide more opportunities for it.

Training with the training.t file revealed a bug in wrapping around, 
previously concealed by the very large sequences in the data (the largest of
which were not being processed.)

Aug 25
Training a 3-layer lstm, I run out of memory at sequence length of 460.
I've been using a max sequence length of 520, but don't need state for
more versions of the hidden state than the length of the sequence.  I'm going 
to attempt to change the memory allocated to the instance, and if that
fails, to create another instance with more memory.  (Which might be a
reasonable time to switch to a cloud instance supporting nvidia... Amazon ec2?)
In order to get a reasonable error, I run the instance with
  sudo sysctl vm.overcommit_memory=2
(otherwise the process is quietly killed.  With overcommit_memory=2 the 
malloc fails, and th gives me a reasonable stack trace, which appears in
a nohup  ... >log output.)
So I edited the instance to get 2CPUs, which I don't need, and 7.5GB, which I
do, and restarted training, again with vm.overcommit_memory=2  
The training still failed, this time at a sequence length of 490 -- which is
where it was killed without traceback if vm.overcommitt_memory was zero.
It's possible that this is an issue with pointer sizes, since the single 
process is very large.  Can torch handle 64-bit pointers?  Possibly also a
google issue.

Torch apparently can't handle more than 4GB; this is apparently due to a 
LuaJit deficiency.  

It seems likely that I can train a 3-layer lstm if I use a max sequence length
of 400.  For a max of 450, I got out-of-memory errors at sequence length of
420, because the maximum batch size was 50, and really used a batch size of 50 
for 420,421, etc. (would probably have used 100+) for these.  Using a max batch ize of 25 permits training to continue; THIS probably would have worked also
for max sequence length of 520.
