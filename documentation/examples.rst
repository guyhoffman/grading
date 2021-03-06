Extended Examples
====================

Individualized Quizzes
------------------------

I decided (for reasons that defy explanation), to give a personalized, self-administered, self-timed, written exam (hello to the other 3 people in the universe with this use case). Canvas, in their infinite wisdom, provide the exact framework for this kind of thing (in the form of graded Quizzes), but then do not allow a file upload to the quiz.  So, we go the convoluted route:  use a Quiz to deliver the exam to students and record a start time, and then use an assignment to collect the responses.  Compare quiz start times to assignment submit times to check whether the students exceeded their allowed time window, and then grade the submissions as usual.  All that remains is setting up a separate quiz for each student and assigning each one to *only* the intended student.  Here we go:

#. First we set up a single assignment to collect the results.  This can be done manually via the web interface or through the API, but in either case, note the assignment id and url so you can link directly to it from the quizzes (not strictly necessary, but makes things easier for the students).

#. Write your bank of exam questions. I decided that I wanted groups of similar questions, wanted to give each student 3 questions to solve, and had ~45 students, so I ended up with 3 groups of questions of 4, 4, and 3 questions in each (48 unique combinations).  Each question lives in its own file named ``prob1.tex``, ``prob2.tex``, etc., and there's a ``header.tex`` in the same directory with all the front matter of the exam (including a ``\begin{enumerate}`` directive). We're going to generate and compile a unique pdf tagged with each student's netid - here I'll just show one instance, but you wrap everything in a loop to cover all students.

    .. code-block:: python

        from cornellGrading import cornellGrading
        from datetime import datetime, timedelta
        import numpy as np
        import subprocess

        c = cornellGrading()
        coursenum = ... # your course number here
        c.getCourse(coursenum)

        prelimpath = ... # path to prelim dir with all the tex files

        # question groups
        block1 = [1,2,3,4]
        block2 = [5,6,7,8]
        block3 = [9,10,11]

        # generate unique combinations and randomize order
        combos = []
        for i in range(len(block1)):
            for j in range(len(block2)):
                for k in range(len(block3)):
                    combos.append([block1[i],block2[j],block3[k]])

        combos = np.array(combos)
        rng = np.random.default_rng()
        arr = np.arange(len(combos))
        rng.shuffle(arr)
        combos = combos[arr]

        # create/get destination folder on Canvas for uploads
        prelimfolder = c.createFolder("Homeworks/Prelim", hidden=True)

        #going to do just the first student, but this is where you'd start the loop
        nid = c.netids[0] # netid
        comb = combos[0]  # problem set

        # write exam tex file
        fname = os.path.join(prelimpath, 'prelim_2020_{}.tex'.format(nid))
        with open(fname, "w") as f:
            f.write("\\input{header.tex}\n")
            for p in comb:
                f.write("\\input{prob%d.tex}\n"%(p))
            f.write("\\end{enumerate}\n")
            f.write("\\bigskip\\bigskip This exam is for %s"%(nid))
            f.write("\\end{document}\n")

        # compile exam
        _ = subprocess.run(["latexmk","-pdf",fname],cwd=prelimpath,check=True,capture_output=True)
        pdfname = os.path.join(prelimpath, 'prelim_2020_{}.pdf'.format(nid))
        assert os.path.exists(pdfname)

        prelimupload = prelimfolder.upload(pdfname)
        assert prelimupload[0], "Prelim Upload failed."

    Note that I've used ``latexmk`` (which of course has to be in my path already) to take care of multiple compilation steps, etc.  If all your questions are simple (no cross-references or anything else requiring multiple compilations, then regular ``pdflatex`` should work fine instead).

#. At this point, we've generated a unique exam PDF for our student(s), so now it's time to place it into a timed quiz.  We have to make it a graded quiz so it can show up in an assignment group. Another important caveat here is how Canvas handles question additions to existing quizzes.  If the quiz is already published and you add a question, Canvas requires that you re-save the quiz before the question becomes visible to students.  I have not found a good way of doing this via the API (what the hell, Canvas?), so the best solution is to generate the quiz *unpublished* add all your questions, and then edit the existing quiz object to make it published.  As a final step, we need to add an assignment override onto the quiz assignment to give it a due date, an unlock date, and to assign it to a single student. Again, showing the same single instance, which you'd loop over for all students.

    .. code-block:: python

        # select assignment group for quizzes to go into
        examgroup  = c.getAssignmentGroup("Exams")

        # write the quiz instructions.
        qdesc = (u'<h2>Stop!</h2>\n<p>By accessing this quiz, you are starting your exam,'
                 u' and the three hour window for submission.\xa0 Do\xa0<strong>not</strong>'
                 u' access the quiz before you are ready to begin.\xa0 If your solution is'
                 u' uploaded any time after the 3 hour window has expired, you will receive'
                 u' no credit for your exam.\xa0</p>\n<p>You do not need to submit this quiz.'
                 u' \xa0 All submission should be made to ' #insert link to submission assignment here
                 u' Your submission must be a\xa0<strong>single, clearly legible PDF.'
                 u' \xa0\xa0</strong>Do not submit individually scanned pages or any other'
                 u' format. You will only have one submission attempt, so be sure to check your'
                 u' work carefully before submitting.</p>')

        # set due date and unlock date
        duedate = c.localizeTime("2020-11-13")
        unlockat = c.localizeTime("2020-11-11",duetime="09:00:00")

        # generate quiz
        quizdef = {
            "title": "Prelim Questions for %s"%nid,
            "description": qdesc,
            "quiz_type":"assignment",
            "assignment_group_id": examgroup.id,
            "time_limit": 180, #this is in minutes
            "shuffle_answers": False,
            "hide_results": 'always',
            "show_correct_answers": False,
            "show_correct_answers_last_attempt": False,
            "allowed_attempts": 1,
            "one_question_at_a_time": False,
            "published": False, #super important!
            "only_visible_to_overrides": True #super important!
            }
        q1 = c.course.create_quiz(quiz=quizdef)

        # add the payload to the quiz
        purl = prelimupload[1]["url"]
        pfname = prelimupload[1]["filename"]
        pepoint = purl.split("/download")[0]
        questext = (
            """<p>Your exam can be accessed here:"""
            """<a class="instructure_file_link instructure_scribd_file" title="{0}" """
            """href="{1}&amp;wrap=1" data-api-endpoint="{2}" """
            """data-api-returntype="File">{0}</a></p>\n<p>Don\'t worry if your browser """
            """warns you about navigating away from this page when trying to download the """
            """exam - it won\'t break anything.</p>""".format(pfname, purl, pepoint)
        )
        quesdef = {
            "question_name": "Prelim",
            "question_text": questext,
            "question_type": "text_only_question",
        }
        q1.create_question(question=quesdef)

        # now we can publish
        q1.edit(quiz={"published":True})

        # create assignment override to set due and unlock dates and the target student
        quizass = c.course.get_assignment(q1.assignment_id)
        overridedef = {
            "student_ids":list(c.ids[c.netids == nid]),
            "title":'1 student',
            "due_at": duedate.strftime("%Y-%m-%dT%H:%M:%SZ"), #must be UTC
            "unlock_at": unlockat.strftime("%Y-%m-%dT%H:%M:%SZ"),
        }
        quizass.create_override(assignment_override=overridedef)

    At the end of this process, you will have a quiz that is only assigned to the one student, and which will record the time at which the student accesses the exam PDF.

#. Once the exam period is over, you can now grab the submission times from the upload assignment as usual.  To get the start times, you need to access the quiz submissions.  Based on how we set this up, there should only be one submission per quiz, which makes things easier.

    .. code-block:: python

        subtime = q1.get_submissions()[0].started_at_date

        # if you want to re-localize the time:
        from pytz import timezone
        subtime.astimezone(timezone('US/Eastern'))

#. Here's some more stuff you can do in terms of post-processing.  In this case, I have set up all of the quizzes and the students have completed their exams.  I saved all of the student net ids (in a column labeled ``netid``), assigned questions (for grading purposes), along with the quiz ids (in a column labeled ``quizid``) in a CSV file called ``assigned_questions.csv``.  Now I can use that in order to access all of the individual start times, get all the end times and update the CSV file with this new info.

    .. code-block:: python

        import pandas

        dat = pandas.read_csv(os.path.join(prelimpath,'assigned_questions.csv'))

        # loop through the quiz ids and get the start times
        starts = []
        for qid in dat['quizid'].values:
            print(qid)
            q = c.course.get_quiz(qid)
            subs = q.get_submissions()
            try:
                starts.append(subs[0].started_at_date)
            except IndexError:
                starts.append(None)
        starts = np.array(starts)

        # now get the exam submission times
        # change this to the name of your particular Exam assignment:
        prelim = c.getAssignment('Prelim')
        tmp = prelim.get_submissions()
        subnetids = []
        subtimes = []
        for t in tmp:
            if t.user_id in c.ids:
                subnetids.append(c.netids[c.ids == t.user_id][0])
                if t.submitted_at:
                    subtimes.append(datetime.strptime(
                        t.submitted_at, """%Y-%m-%dT%H:%M:%S%z"""))
                else:
                    subtimes.append(np.nan)
        subnetids = np.array(subnetids)
        subtimes = np.array(subtimes)

        # now calculate each student's test duration
        testtimes = []
        subtimes2 = []
        for j in range(len(dat)):
            try:
                testtimes.append((subtimes[subnetids == dat['netid'].values[j]][0] - starts[j]).seconds/3600)
                subtimes2.append(subtimes[subnetids == dat['netid'].values[j]][0])
            except TypeError:
                testtimes.append(np.nan)
                subtimes2.append(np.nan)
        testtimes = np.array(testtimes)
        subtimes = np.array(subtimes2)

        # add info to CSV and write back to disk
        dat = dat.assign(Start_Time=starts)
        dat = dat.assign(End_Time=subtimes)
        dat = dat.assign(Duration=testtimes)
        dat.to_csv(os.path.join(prelimpath,'assigned_questions.csv'),index=False)

