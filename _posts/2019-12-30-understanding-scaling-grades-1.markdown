---
layout: post
title:  "Understanding Grade Scaling at Purdue"
date:   2019-12-16 10:00:00 -0600
categories: 10 Min Read
---
Scaling/Normalizing grades has never been a very clear me. So this winter, I decided to take a look at the data from the courses I have been a TA for. Most people have the notion that the final grade distribution is roughtly a [Normal](https://en.wikipedia.org/wiki/Normal_distribution) distributed about the mean, hence the name normalization or scaling. At this point I should point out at Purdue, scaling and normalization are used interchangably, but they are slightly different. Simplest way to explain this is to say normalization can change the shape of the curve but scaling merely changes the scales. So when professors say scaling I believe they mean normalizing. However I have found these two terms used so inter-changeably that it best to decide the meaning depending on the context.

Lets look at the final grades for a course. I will use panadas and matplotlib for data processing and vizualization.
{% highlight python %}
import pandas as pd
import matplotlib.pyplot as plt
df = pd.read_csv('/home/anirban/Projects/gradebook/172_fall.csv') 
grades=df['COURSE TOT [Total Pts: up to 2,000 Score] |4336090'].str.replace(',', '').astype(float)
#Scale grades
grades=grades/2000.0
#Make a histogram
n, bins, patches = plt.hist(grades ,bins=100, facecolor='blue', alpha=0.75)
plt.xlabel('% Marks')
plt.ylabel('# of students')
plt.show()
{% endhighlight %}

These scores are already curved. This is what the histogram looks like. 
![Normalized Score Histogram]({{site.url}}{{site.baseurl}}/images/gradebook/norm_cum_sc.png)
Hmm... that does not look like a Gaussian at all. Maybe, because the scores are already normalized, we get such a skewed distribution. Lets then look at the raw totals from labs, recitation and exams. 
{% highlight python %}
labAvg = df['LAB-Avg [Total Pts: up to 20 Percentage] |4332915'].astype(float)/100.0
recAvg = df['REC-Avg [Total Pts: up to 25 Percentage] |4332917'].astype(float)/100.0
lecAvg = df['LEC-Avg [Total Pts: up to 4 Percentage] |4332923'].astype(float)/100.0
hwAvg = df['HW - Total [Total Pts: up to 300 Score] |4050147'].astype(float)/300.0
quizAvg = df['QUIZ - Total [Total Pts: up to 300 Score] |4050148'].astype(float)/300.0
examAvg = df['ALL EX: Max Total [Total Pts: up to 800 Score] |4328053'].astype(float)/800.0
#Scale the total
total = (labAvg+recAvg+lecAvg+hwAvg+quizAvg+examAvg)/6.0
n, bins, patches = plt.hist(total ,bins=100, facecolor='blue', alpha=0.75)
plt.xlabel('% Marks')
plt.ylabel('# of students')
plt.show()
{% endhighlight %}
We get 
![Raw Score Histogram]({{site.url}}{{site.baseurl}}/images/gradebook/cum_sc.png)
That looks even worse (I guess good or bad depends on your point of view. A student should be very thrilled with this). With a raw score distribution like this, there is no obvious way of saying who put more effort in the course. Also it is a nightmare to normalize curves like this.

However, this distribution should not come as a surprise. The labs and recitations gave full points to a student just for being present. Hence almost everyone got 100% for those. So, my guess is that HWs and Exams should show a more Normal distribution. The plots we get are 
![Exam and HW Histogram]({{site.url}}{{site.baseurl}}/images/gradebook/exam_hw_raw.png)
The exam distribution is kind of better. For HW again looks almost like a flipped Poisson Distribution. If you dont know about [Poisson Distribution I would highly encourage you to read about it](https://en.wikipedia.org/wiki/Poisson_distribution). I like to think of the flipped Poisson curve this way. Say we had a God who was handing out free answers. Everyone would want that. But people are lazy and sometimes other things can come in the way. So lets say the probability that a student gets answer for a particular HW problem wrong is about 2% or 0.02. So on average the students gets 98% problems correct. Only a few gets a significant number of problems wrong. 


While this is somewhat good approximation, the original distribution has a much longer tail. So our model is not as good. I will try to make a follow up post on this. At this point however I was curious to see if these skewed distributions are just peculiar to this course or a general characterestic. Fortunately I have dataset for another course.
Hmm... that does not look like a Gaussian at all. Maybe because the scores are already normalized we get such a skewed distribution. Lets then look at the raw totals from labs, recitation and exams. 
{% highlight python %}
a = df['Weighted Total [Total Pts: up to 179.7 Percentage] |4388776']/100.0
labAvg = df['Total Lab [Total Pts: up to 130 Score] |4712224'].astype(float)/130.0
recAvg = df['Total Rec [Total Pts: up to 130 Score] |4712221'].astype(float)/130.0
lecAvg = df['Total Lec [Total Pts: up to 52 Score] |4712222'].astype(float)/52.0
hwAvg = df['Total HW [Total Pts: up to 254 Score] |4712216'].astype(float)/254.0
examAvg = df['Total Evening Exams [Total Pts: up to 200 Score] |4712229'].astype(float)/200.0
#Scale the total
total = (labAvg+recAvg+lecAvg+hwAvg+quizAvg+examAvg)/5.0
plt.subplot(221)
n, bins, patches = plt.hist(a ,bins=100, facecolor='blue', alpha=0.5)
plt.xlabel('% Marks')
plt.ylabel('# of students')
plt.title('Normalized Total Score')
plt.subplot(222)
n, bins, patches = plt.hist(total ,bins=100, facecolor='blue', alpha=0.5)
plt.xlabel('% Marks')
plt.ylabel('# of students')
plt.title('Raw Total Score')
plt.subplot(223)
n, bins, patches = plt.hist(hwAvg ,bins=100, facecolor='blue', alpha=0.5)
plt.xlabel('% Marks')
plt.ylabel('# of students')
plt.title('HW Total Score')
plt.subplot(224)
n, bins, patches = plt.hist(examAvg ,bins=100, facecolor='blue', alpha=0.5)
plt.xlabel('% Marks')
plt.ylabel('# of students')
plt.title('Exam Total Score')
{% endhighlight %}
and here are the plots 
![Course2 Histogram]({{site.url}}{{site.baseurl}}/images/gradebook/course2_plots.png)

Plots for this course is not much skewed given this course is a higher level course than the first one. The most stark difference is in the exam plots where we clearly see some sort of normal distribution. The exams for this course had a subjective part and a multiple choice part. Plotting the two seperately shows the most stark contrast.
{% highlight python %}
subjective = df['Exam 1 HG [Total Pts: 30 Score] |4531308'].astype(float) + df['Exam 2 HG [Total Pts: 30 Score] |4681344'].astype(float)
multipleChoice = df['Exam 1 MC [Total Pts: 70 Score] |4531196'].astype(float) + df['Exam 2 MC [Total Pts: 70 Score] |4681340'].astype(float)
subjective = subjective/60.0
multipleChoice = multipleChoice/140.0
n, bins, patches = plt.hist(subjective ,bins=30, facecolor='blue', alpha=0.25, label='Subjective')
n, bins, patches = plt.hist(multipleChoice ,bins=30, facecolor='red', alpha=0.25, label='Multiple Choice')
plt.xlabel('% Marks')
plt.ylabel('# of students')
plt.legend()
plt.title('Subjective vs Multiple Choice Score')
plt.show()
{% endhighlight %}

![Course 2 MC vs HG]({{site.url}}{{site.baseurl}}/images/gradebook/course2_subjectiveVsMc.png)

The gaps in the Multiple Choice part is due to discreet possible grades. The hand graded portion looks much more similar to the distribution I had in mind before I started this post. The overall trend is still same as the first course. The underlting theme of all the plots seems to be a slow rise and then quicker falloff. We will get into the distrubtion in the second post. 
Lastly I wanted to see if there was correlation between exam scores. Each of the courses had multiple exam and I would normally expect to see quite bit of correlation between the scores. I will plot the Subjective parts only since it is continious, unlike the Multiple choice part which is discreet, and also more normal. 

![Course 2 HG vs HG]({{site.url}}{{site.baseurl}}/images/gradebook/course2_subjectiveVssubjective.png)

This was a bit disappointing for me since the correlation is not very strong. But that is probably because the curves for the two subjective parts are significantly different. But on the bright side, the students performed clearly better in the second exam (since I remember both of the subjective parts being more or less of the same level)

I would like to end this post on the note that, in a world where curves look like this, what significance does grade B over A hold ? Does it really tell us what people think a B over A would tell you about the student ? Can we make these curves more non-skewed ? Yes, simply by making the courses more difficult. But should we ? 


[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
