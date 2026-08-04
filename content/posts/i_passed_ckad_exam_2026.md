+++
date = '2026-08-04T15:16:15-05:00'
draft = true
title = 'I Passed the CKAD Exam'
tags = ["kubernetes", "ckad", "homelab"]
categories = ["certs"]
description = "How I prepped for and passed the Certified Kubernetes Application Developer (CKAD) exam with KodeKloud, Killer.sh, and KillerCoda."
layout = "post"
+++


# I Passed the CKAD

Welcome back, friend. I passed the Certified Kubernetes Application Developer (CKAD) exam, the second certification in my push toward the full Kubestronaut set. I finished CKA in early July, and rather than take a real break, I rolled almost straight into CKAD prep while everything was still fresh. That turned out to be one of the better calls I've made in this whole process. Let me walk through how the prep went, how exam day felt this time around, and what's next.

## The Prep

I followed almost the same playbook as CKA: a KodeKloud course built specifically for CKAD, the KillerCoda scenarios, and a Killer.sh simulated exam. Because I'd already paid for KillerCoda access during CKA prep, I still had it available and used the [CKAD-specific course](https://killercoda.com/course/ckad) there instead of paying for it twice. On the KodeKloud side I worked through their [core CKAD course](https://learn.kodekloud.com/learn/courses/certified-kubernetes-application-developer-ckad) and dipped into their [Ultimate CKAD Mock Exam Series](https://learn.kodekloud.com/learn/courses/ultimate-certified-kubernetes-application-developer-ckad-mock-exam-series), though I only ended up completing four of the eight mock exams in that series rather than working through all of them. I also took just one Killer.sh CKAD simulated exam instead of two, since both versions of that simulator are identical and running the same exam twice didn't seem like it would teach me anything new.

Total prep time came out to about three weeks, though a job interview landed in the middle of that window and ate into a few days I'd otherwise have spent studying. It went well, though I'll save the details for another time. I could probably have finished in two weeks without that detour, but between the interview and choosing to take things a little easier than I did for CKA, three weeks is where it landed. Having taken CKA so recently, on July 3rd, that material was still fresh in my memory, and a fair amount of it turned out to be directly relevant to CKAD. Because of that, I honestly didn't find myself leaning on the docs or `kubectl explain` nearly as much as I did during CKA.

I kept leaning on the same hands-on habits from CKA prep too: running scenarios against real clusters rather than just watching videos, and pulling exercises from [kubernetes-to-the-moon](https://github.com/mrmcmuffinz/kubernetes-to-the-moon), the free practice repo I built with an AI assist while studying for CKA. I wrote more about how that repo came together, and about the [Raspberry Pi cluster](/posts/rpi_bootstrapping/) some of its scenarios run against, in my [CKA writeup](/posts/i_passed_cka_exam_2026/), so I won't repeat all of that here. It was just as useful the second time through.

## Exam Day

Exam day itself was smoother than CKA's was. I can't talk about the actual exam content, that's against the Linux Foundation's terms of service, but I can say the testing platform behaved itself this time. No disconnects, no lag, no weird paste issues, which is not something I take for granted after my own portal reset during the Killer.sh simulator for CKA. I went in with the same time-management approach that worked for CKA: budget roughly a set number of minutes per question, mark anything that's eating too much time and come back to it later, and don't let one stubborn problem burn the clock on everything after it. That approach carried over cleanly and I didn't need to relearn it under pressure.

## The Results

Results took less time to come back than CKA's did. I passed with a 94%, well clear of the 66% needed to pass and past the 90% target I'd quietly set for myself after finishing CKA at 86. If I'm being honest about the full picture, this wasn't actually my first run at CKAD. I sat this exam once before, back in 2018, and didn't pass it then. Coming back eight years later and clearing it with room to spare felt like closing a loop I'd mostly forgotten was still open.

You can see the fuller picture in my exam history below: this year's CKAD pass, the CKA pass from July, and that 2018 attempt still sitting there as a reminder of where I started.

![Linux Foundation exam history showing the CKAD pass at 94%, the CKA pass at 86%, and an earlier CKAD attempt from 2018 that did not pass](/img/i_passed_ckad_exam_2026/2026_abraham_cabrera_exam_scores.png)

And here's the certificate itself, fresh from the Linux Foundation, signed off the same day I sat the exam.

![Certified Kubernetes Application Developer certificate showing a passing score of 94%](/img/i_passed_ckad_exam_2026/2026_abraham_cabrera_ckad_certificate.png)

## What's Next

Next up is the Certified Argo Project Associate, or CAPA. I already have the voucher in hand, and if I'm honest, the deciding factor wasn't some grand plan about which cert teaches me the most right now. It's that CAPA's voucher expires sooner than anything else I'm holding, and I'd rather use it than let it lapse. That's not to say there isn't real interest there too, since Argo CD, Workflows, and Events are already running in my homelab, so there's natural overlap with what I've been building anyway. The Certified Kubernetes Security Specialist is still on the list after that; I haven't dropped it, it's just been pushed further out by the calendar rather than by choice.

Well, that's all. See you in the next one.

**P.S.** I used Claude to help build the study plan for this one, the same way I did for CKA, and used it again afterward as a sounding board while writing this post. The ideas, the prep decisions, and the final edits are still mine.
