all:	data_LAV training testform.txt

training:	task2-train.txt split-task.py
	python3 split-task.py <task2-train.txt

data_LAV:	training
	python3 xliterate.py -infile training

# 'training' is a proxy for 'testing' file
#  and testform.py always write test.txt file
test.txt:	training
	python3 testform.py <testing


#
%.stat:	cv/%.t7
	th test.lua -init_from $^ < varDialTrainingData/test.txt | (cd varDialTrainingData; th statistics.lua )

wordstats:	wordfreq.py
	python3 wordfreq.py | (cd varDialTrainingData; th statistics.lua )
