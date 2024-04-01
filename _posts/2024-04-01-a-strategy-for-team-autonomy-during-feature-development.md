---
layout: post
title:  "A strategy for team autonomy during feature development"
date:   2024-04-01 00:00:00 +0100
categories: ways-of-working
---

![FeatureDevelopment](/images/2024-04-01-a-strategy-for-team-autonomy-during-feature-development/feature-development.png)


As a tech lead I consider that one of my main goals is to help my teammates to become autonomous and capable of making informed decisions without having to run everything by me. A tech lead should be an enabler and a force-multiplier instead of a bottleneck or a knowledge silo.

In the past I failed in doing this and I deeply felt the implications of it. This was specially critical during product feature development. Some of the implications I felt were:
- Lack of understanding of the product as a whole by the team.
- Work moved slower if I was not around to provide feedback or guidance.
- Problems were identified in later phases of development due to lack of knowledge of the team regarding the product being built.
- Any obstacles that arose had to come to me since the team lacked the autonomy to make informed decisions.

This got me thinking on why was this happening and how could I fix it and make the team autonomous and better prepared to face unforeseen challenges. To describe this situation I first have to briefly describe the [modus operandi](https://pt.wikipedia.org/wiki/Modus_operandi) of my current company when approaching new feature development. It goes as follows:
1. First we have our product owners talk with the stakeholders and understand what are the features that could bring higher business value.
2. After those features are identified we have the principal engineers and the tech leads of the teams defining a high level solution, identifying possible dependencies or obstacles, and providing a high level estimation for the work at hands.
3. After that we have the tech leads defining a detailed solution, making the work breakdown and helping our delivery managers to draw a viable delivery plan.
4. Finally the work is then explained and assigned to the team that will develop it, which is then responsible for delivering it according to the defined delivery plan.

Up until recently what I did as a tech lead was to design a detailed solution, breakdown it down in small easily understandable pieces of work and explaining them to the team. When the work got to the team everything was already analyzed and defined and not much work was needed besides the implementation itself.

After some time I began to understand that this was not the right approach. There were a lot of small indications of that but the most important one was that I noticed that the team didn't have a high level understanding of the feature being built. This was visible both from a business as well as a technical perspective.

Of course I did sharing sessions explaining the system and how everything fitted together but that wasn't enough. This wasn't enough because they didn't had the necessary context to think about the assumptions, tradeoffs and overall implications of the design solution. And of course this was not their fault. **It was mine that created a knowledge silo when defining and detailing the solution on my own.**

After understanding this I tried to come up with a better strategy to achieve team autonomy during feature development. The proposed strategy goes as follows:
  - For every feature requirement that the product team defines write a very brief description of the technical work that must be done to achieve it.
  - Split the feature into a few independent work streams. 2 or 3 would be ideal.
  - Assign each one of the workstreams to a separate team member and let it come up with a detailed solution (or possible solutions and correspondent tradeoffs) for it. Make them do a work breakdown for it as well. At this point you should define a time window for that work to be concluded.
  - Do recurrent check points with those team members to provide quick feedback on the work and prevent them from being overwhelmed or from walking too long on the wrong direction. These check points should included each work stream owner in order to guarantee consistency in the final solution.
  - After that, gather the team, review each work stream and collaboratively draw a final solution and work breakdown.

Benefits of the proposed strategy:

  - By defining the solution for the problem at hands and consider its tradeoffs and overall implications the team gains a comprehensive understanding of the feature under development.
  - By grasping the feature at a high level, the team's autonomy experiences a significant boost. This empowers them to make informed decisions and take ownership of their tasks more independently.
  - Team members become more engaged as they feel valued, knowing their opinions are considered and their voices are heard in the decision-making process.
  - Empowering the team to devise solutions challenges them and fosters growth, particularly beneficial for less experienced teams. Rather than spoon-feeding solutions, this approach encourages them to think critically and innovate.
  - Taking ownership of the solution instills a sense of self-accountability within team members, leading to an overall improvement in solution quality.
  - By empowering the team to be autonomous and independent, as a tech lead, you can allocate time to focus on other critical tasks such as defining roadmaps or enhancing services and processes.

Challenges and points to consider:

  - Understand how the team feels about this approach. While some may be enthusiastic, others might not be as receptive. It's crucial to identify team members who are willing to participate in such initiatives.
  - Bring the team together to assess individual work streams. The final solution should be unified and coherent. Facilitating communication is key to achieving this coherence.
  - As a tech lead, ensure that team members have the necessary support. Avoid assigning challenges that could induce nervousness or anxiety. Instead, assure them that you have their back in case they encounter any issues they cannot resolve.
  - Have regular meetings with each team member to give feedback regularly. Don't wait until the end to provide feedback, as it might require big changes. Giving feedback often helps improve work steadily and reduces the need for big changes later.

After implementing this I noticed that team involvement and ownership skyrocket. Although I learned this the hard way I feel that now I value team work even more. I'm writing this with the hope of helping anyone in a position of technical leadership to achieve the benefits of a strategy such as this without having to go through the obstacles I went through :)

Cheers!
