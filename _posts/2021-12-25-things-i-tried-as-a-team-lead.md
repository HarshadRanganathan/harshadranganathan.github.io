---
layout: post
title: "Things I Tried As A Team Lead"
date: 2024-07-25
excerpt: "Opinionated article on the things I tried as a Team Lead - what worked and what didn't work"
tag:
- Team Lead
comments: true
---

## Documentation: Why It Matters and How to Make It Work

Documentation is crucial for any team. It keeps information accessible, helps understand system workings, and guides code execution and troubleshooting. 📚

However, managers often ask:
- Is it documented?
- Can we document it so everyone knows?

This can shift focus away from solving the real problem.

Even with documentation, issues can arise:
- Lack of clarity
- People not using it
- Documentation is hard to find
- Not enough time spent on it

Despite these challenges, documenting is key. It lets you confidently say, "Yes, it's documented," and directs attention back to solving the problem.

Without documentation:
- Team members waste time figuring things out
- You answer the same questions repeatedly

Common challenges with documentation include:
- Time constraints
- Assumptions about what’s known
- Not understanding user perspectives

Keep your docs clear and accessible to avoid these pitfalls and streamline problem-solving. 🌟

Things that I implemented in my team:

### Choosing the Right Documentation Platform

Whether your team uses Microsite, SharePoint, or Confluence, centralizing your documentation is key. I prefer using Microsites created with static site generators like MkDocs or Docusaurus for several reasons:

- **Code Repository Integration:** Docs are maintained in a code repo, similar to how engineers manage README files.
- **Consistency:** Engineers are already familiar with markdown, so it's an easy transition.
- **Tool Changes:** Switching tools (e.g., from Confluence to SharePoint) can be messy, especially with vendor-specific plugins like draw.io. Static sites avoid these issues.
- **Review and Feedback:** Documentation can be reviewed through pull requests (PRs), allowing for suggestions and improvements.
- **Guidelines:** Establish general documentation guidelines with PR templates.

Using static site generators keeps your documentation organized and adaptable to tool changes. 🌐📄

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/aws-platform-docs.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2022/05/aws-platform-docs.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2022/05/aws-platform-docs.png">
            <img src="{{ site.url }}/assets/img/2022/05/aws-platform-docs.pngg" alt="">
        </picture>
    </a>
</figure>


#### Documentation Guidelines for the Team

Here are some guidelines to make our documentation more effective:

- **Use Concise Headings:** Keep headings clear and to the point.
- **Incorporate Searchable Keywords:** Include terms like exception messages in troubleshooting guides to make searches easier.
- **Provide Code Examples:** Include code snippets for clarity.
- **Use Syntax Highlighting:** Enhance readability with code syntax highlighting.
- **Add Screenshots:** Use screenshots for visual guidance, especially for complex steps like database connections.

Screenshots help some users understand better, but remember to update them as tools change. Always balance between text and visuals to ensure documentation remains clear and searchable. PR reviews can help maintain this balance.

**Additional Tips:**

- **Be Detailed:** Avoid assumptions and provide thorough information.
- **Preview Changes:** Ensure proper formatting, image loading, and styling before finalizing.
- **Check for Clarity:** Review your documentation as if you're a new or external team member.
- **Use Relevant Keywords:** This helps others find your document easily and reduces the need for follow-up queries.

**Challenges:**

- **Ongoing Practice:** Engineers need practice to follow these guidelines effectively.
- **Different Writing Styles:** Allow for individual styles but maintain overall clarity.
- **Training for New Writers:** Offer guidance and suggestions in PRs to help improve documentation skills.

Following these guidelines will help make our docs more useful and user-friendly. 📚📝

{% include donate.html %}
{% include advertisement.html %}

### Docs Backlog Management

Create a backlog to keep track of documentation needs, including:

- FAQs
- Pending Documentation
- Architecture Designs
- Guideline Documents
- Updates for Existing Docs
- Runbooks
- Troubleshooting Guides for New Issues

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/github-issues-doc-backlog.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/github-issues-doc-backlog.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/github-issues-doc-backlog.png">
            <img src="{{ site.url }}/assets/img/2021/12/github-issues-doc-backlog.png" alt="">
        </picture>
    </a>
</figure>


**Challenges:**

- **Team Participation:** Ensuring all team members contribute to the backlog requires commitment.
- **Regular Reminders:** Use standups to remind team members to add resolved issues and requests for clarity to the backlog. This minimizes repeated answers and saves time on support.

A well-maintained backlog helps keep documentation comprehensive and up-to-date. 📋🛠️

### Weekly Docs Contribution

To ensure regular documentation updates, we encourage team members to contribute to documentation as they work on related tasks. However, this often doesn’t happen due to busy schedules and delivery pressures.

To address this, we implemented a weekly documentation contribution schedule:

- **Day Chosen:** `Friday` is designated for documentation work. It’s a day when team members are winding down and can dedicate an hour or so to this task.

**How We Enforce It:**

1. **Add Reminder:** Create a reminder card in the Teams planner tab for "Friday Docs," including due dates.
2. **Thursday Standup:** Review the docs backlog list, identify priority items, and select quick wins to document.
3. **Team Discussion:**
   - Assign documentation tasks to each member
   - Ensure sufficient details are available
   - Clarify any doubts
4. **Friday Contribution:** Team members spend time on documentation and raise pull requests (PRs).

This practice has been effective for regular documentation updates.


<figure>
    <a href="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
            <img src="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png" alt="">
        </picture>
    </a>
</figure>

**Problems Faced:**

1. **Reminder Management:** The standup facilitator needs to bring up the reminder card each Thursday.
2. **Incomplete Contributions:** Sometimes, team members still miss their documentation tasks, requiring follow-up reminders for the next week.

{% include donate.html %}
{% include advertisement.html %}

### External Contributors

When your team is small or there's a high volume of documentation, involving external contributors can help scale up the effort.

**Approaches to Involve External Contributors:**

- **Promote Learning Opportunities:** Highlight the chance for them to work with your team and learn about the documented topic.
- **Offer Incentives:** Recognize their contributions with thank you points, acknowledgment in emails, or shout-outs in meetings.
- **Avoid Hard Deadlines:** Allow flexibility in delivery to accommodate their schedules.

**Considerations:**

While working with external contributors may require additional time for explaining processes, providing walkthroughs, and supporting them with necessary details, it can effectively boost your documentation efforts.

### PRs

- **Update README:** If a pull request (PR) does not include updates to the README (when applicable), request that the README be updated before approval.
  
- **Track Pending Changes:** For urgent PRs that require immediate approval, ask the team member to raise an issue in the repository to track the pending documentation updates for later.

{% include donate.html %}
{% include advertisement.html %}

## Daily Standup

- **Facilitator Assignment:** Assign a standup facilitator or rotate the role among team members weekly or per sprint.
  
- **Round Table Updates:** Ensure each team member gives a quick update. If someone starts to diverge, remind them to use the parking lot for detailed discussions to keep the standup focused.

- **Time Management:** Keep the standup within the allotted time. If more discussion is needed, decide whether to continue, take a short break, or schedule a separate meeting.

- **Team Lead Role:** Use the standup to provide:
  - General updates
  - Status checks on specific work items
  - Brief discussions on specific topics

### Standup Cards

To keep track of topics to raise, use the Teams Planner tab to create "Daily Standup Items" cards. Team members and leads can add cards with due dates whenever they think of something.

**Process:**
- **Card Creation:** Add items to the Daily Standup Items tab as they come up.
- **Card Review:** During the standup, review these cards and allow the creator to provide updates or ask questions.

This method helps ensure that no important topics are missed during the standup.

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
            <img src="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png" alt="">
        </picture>
    </a>
</figure>

**Problems Faced:**
- **Integration:** Ensure the card process is consistently followed during standups.
- **Card Volume:** Prioritize important cards for discussion and consider delaying less critical ones.
- **Personal Notes:** Some team members may prefer using their own notes, so be flexible in accommodating different methods.

{% include donate.html %}
{% include advertisement.html %}

## Retrospectives

Use a tool of your choice to manage retrospectives with columns such as **Start**, **Stop**, **Continue**, and **Action Items**.

**Process:**

1. **Facilitator Rotation:** Rotate the retro facilitator every sprint.
   
2. **Populate Cards:** Start a timer for 10-15 minutes for team members to add cards to each section.

3. **Review Cards:** The facilitator reviews the cards, merges duplicates, and seeks clarity on items that need it.

4. **Voting:** Start a timer for 5 minutes to vote on cards. Each team member has a set number of votes to prioritize important issues.

5. **Generate Action Items:** The facilitator works with the team to create actionable items and assign ownership.


<figure>
    <a href="{{ site.url }}/assets/img/2021/12/retro-board.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/retro-board.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/retro-board.png">
            <img src="{{ site.url }}/assets/img/2021/12/retro-board.png" alt="">
        </picture>
    </a>
</figure>

**Common Challenges:**

- **Non-Actionable Items:** Some cards may require more authority to address.
- **Excuses for Inaction:** Assigned team members might cite busy schedules or forgetfulness.
- **Unaddressed Cards:** Cards without assigned action items may feel ignored.
- **Ventilation vs. Resolution:** Retrospectives can become a space for complaints rather than solutions.

**Solutions:**

- **Address All Cards:** Even if a card doesn’t get many votes, if it’s important for any team member, take time to address it or create an action item.
- **Reasonable Action Items:** Focus on items that can be resolved by the team or manager. Raise issues requiring higher authority as feedback to management.
- **Retro Checkpoints:** Add a card to Daily Standup Items with a due date set for mid-sprint. Use it to review progress on action items and remind team members of pending tasks.


<figure>
    <a href="{{ site.url }}/assets/img/2021/12/teams-planner-card-retro-checkpoint.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-retro-checkpoint.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-retro-checkpoint.png">
            <img src="{{ site.url }}/assets/img/2021/12/teams-planner-card-retro-checkpoint.png" alt="">
        </picture>
    </a>
</figure>

{% include donate.html %}
{% include advertisement.html %}

## Backlog Grooming

**Best Practices:**

- **Frequent Groomings:** Hold weekly grooming meetings or break them into smaller sessions based on team feedback to avoid a single large call.
  
- **Encourage Team Input:** Foster discussions rather than solely providing details yourself. Create an environment where team members feel comfortable asking for clarification or admitting they don’t understand.

- **Round Table Participation:** If some team members are not contributing, do a round table to ensure everyone has the opportunity to provide input, raise questions, or give feedback.

- **Detail Adequacy:** Ensure each story has enough detail for any team member to understand and work on it.

- **Story Points:** Use tools to gather story point estimates from all team members. When in doubt, lean towards a higher estimation rather than a lower one.

- **High Estimations:** If a team member provides a very high estimate, ask for justification to determine if it’s reasonable. Consider splitting large stories into smaller ones if needed.

- **Spikes:** For new stories, check if a spike (research or exploration task) is needed to understand feasibility, approach, and potential challenges. Conduct this spike in the sprint before the implementation to ensure readiness.

- **New Team Members:** Ask new members to review stories during the week and come prepared with questions for the next grooming session.

**Problems Faced:**

- **Team Participation:** Engaging all team members can be challenging and requires their commitment.
  
- **Uncertainty and Questions:** Some members may ask many questions or claim they don’t understand what needs to be done after picking up a story.

- **Assumptions:** Team members might assume they won’t work on certain stories and therefore not concern themselves with story details.

- **Experience Gap:** Assessing impacts and risks may require more experience or domain knowledge, which can affect story evaluation.

{% include donate.html %}
{% include advertisement.html %}

## PI Planning

**Best Practices:**

- **Use Whiteboarding Tools:** Leverage digital whiteboarding tools to plan work items across sprints. This helps visualize and organize tasks effectively.

- **Gather Team Inputs:** Solicit input from team members to identify and allocate technical debt or backlog items they want to address in the upcoming Program Increment (PI). Check if these can be accommodated.

- **Include Focus Areas:** Add sections for different focus areas, such as CI/CD, security, etc., to ensure all critical aspects are covered.

- **Prioritize the Backlog:** Review and prioritize the backlog to identify potential items that can address gaps in the PI planning.

- **Publish Planning:** Share the planning with dependent teams to review and raise any missing items or concerns.

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/pi-planning-board.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/pi-planning-board.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/pi-planning-board.png">
            <img src="{{ site.url }}/assets/img/2021/12/pi-planning-board.png" alt="">
        </picture>
    </a>
</figure>


{% include donate.html %}
{% include advertisement.html %}
## Task Assignment

**Ideal State:**

- **Self-Assignment:** Team members choose tasks as they complete previous ones.
- **Support Requests:** Team members promptly indicate if they are blocked or need additional support.
- **Priority Awareness:** Team members are aware of and work on priority tasks.

**Reality:**

- **Waiting for Assignments:** Team members often wait for tasks to be assigned.
- **Delayed Support Requests:** They may delay mentioning support needs until asked.
- **Blocked Communication:** Team members might wait for the standup to report blocks instead of using group chat.
- **Priority Confusion:** Team members might work on low-priority items without knowing the priorities.

**What I Did:**

- **Explicit Task Assignment:** Started by assigning tasks explicitly to team members while fostering a culture where they pick tasks based on priorities.
- **Support and Task Swapping:** If a team member struggles or delays, check if they need more support or consider swapping tasks.
- **Prioritize Work Items:** Organize the sprint board with tasks sorted by priority so team members pick from the top.
- **Weekly Priority Calls:** Hold a weekly Monday call to clarify priority items if they change frequently.

**Problems Faced:**

- **Task Disinterest:** Some team members may avoid certain tasks (e.g., boring or trivial ones). Remind them that all tasks are part of the job.
- **Competing Interests:** Multiple team members may be interested in the same task. Assign based on factors like priority and past assignments.

{% include donate.html %}
{% include advertisement.html %}

## Design Discussions/Spikes

**Best Practices:**

- **Evaluate Approaches:** When team members propose using new or trending tech, ask them to evaluate various approaches. They should provide a list of options, including pros and cons, and make a recommendation.

- **Prepare for Meetings:** Schedule a team meeting within an agreed time box. Share the analysis report with the team in advance so everyone can review and prepare.

- **Team Evaluation:** In the meeting, review the proposed approaches and recommendations. Discuss any new inputs, risks, or impacts and decide if further analysis is needed.

- **Focus on Incremental Improvements:** Prefer incremental changes over large, disruptive ones.

- **Avoid Over-Engineering:** Assess if the proposed solution is unnecessarily complicated or over-engineered.

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/spike-analysis-template.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/spike-analysis-template.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/spike-analysis-template.png">
            <img src="{{ site.url }}/assets/img/2021/12/spike-analysis-templates.png" alt="">
        </picture>
    </a>
</figure>

**Problems Faced:**

- **Theoretical vs Practical:** Challenges may arise when theoretical analysis doesn’t align with practical constraints.
  
- **Delivery Constraints:** Delivery timelines or constraints may limit the options available for implementation.

{% include donate.html %}
{% include advertisement.html %}

## Technical Backlog

- **Prioritize Items:** Organize backlog items by priority so you can quickly address any tech debt or items the team wants to work on to fill sprint gaps.

- **Buffer for Tech Debt:** Collaborate with the manager or product owner to include a buffer in the sprint to accommodate technical debt.

## JADs (Joint Application Design Sessions)

- **Set Agenda:** Prepare and share the agenda in advance.

- **Small Group Size:** Keep the group small to facilitate decision-making and avoid lengthy discussions.

- **Time Box:** Limit the session duration to ensure discussions remain focused and productive.

- **Capture and Store Notes:** Document meeting notes and decision points, and store them in a code repository or documentation for future reference.

- **Manage Scope:** Politely interrupt to address off-topic discussions outside the meeting.

- **Prepare Recommendations:** It’s often more effective to present recommendations rather than expecting solutions to emerge during the meeting.

- **Ensure Alignment:** Discuss with team members beforehand to ensure everyone is on the same page and avoid conflicting pitches.

- **Influence the Direction:** Guide the meeting to stay on track and achieve desired outcomes.

## Priorities

- **Clearly Indicate Priorities:** Communicate priorities to the team clearly.

- **Order Sprint Items:** Arrange sprint items by priority so team members tackle high-priority tasks first.

- **Support for Priorities:** Monitor team members’ progress with priority items and provide support as needed to prevent delays.

- **Update on Changes:** Inform the team promptly when priorities shift.


## Team Communication Model

Sometimes you have conflicts within the team as to how the internal communication needs to take place.

- Some prefer not to have personal pings
- Some prefer to have everything discussed in a team channel so that every conversation is visible to every team member so that they are kept in loop as well
- Some prefer not to jump on calls

It's important to have a meeting within the team to discuss each person's thoughts and reach a collective communication model based on a majority.

e.g.

||When to use|What to avoid|Examples|
|--|--|--|--|
|Teams Chat Room| General discussions<br/>Support questions<br/>Quick PR approvals| Lengthy discussions<br/>Mixing different topics when a conversation is already happening| |
|Dedicated Chat Room |Specific work items e.g. Spike | |DB Data Migration<br/>Production Release<br/>EKS Upgrades|
|Teams Channel |Announcements<br/>Questions which may not get quick answers<br/>Maintain discussion thread for easy reference | | Security Remediations<br/>Docs<br/>General announcements<br/>Sprint reminders|
{:.table-striped}

{% include donate.html %}
{% include advertisement.html %}


## PRs

**Typical Problems:**

- PRs are not reviewed in a timely manner.
- PR changes are too large.
- PR review comments are ignored.
- Too many changes proposed.
- Rework of PR needed.
- Tone of review comments.
- Team members expect their PRs to be approved without approving others.
- PRs sit for more than a week.
- Changes suggested in one place are not reflected elsewhere.
- Instead of responding to PR comments, you receive pings with explanations.

**Recommendations:**

- **Daily Review Time:** Dedicate time each day to review pending PRs.
  
- **Reminders:** If awaiting PR approval, post reminders in the team channel or raise it during standup if it’s a blocker.

- **Smaller Chunks:** Break PRs into smaller, manageable chunks for easier review. Avoid mixing unrelated changes.

- **Provide Descriptions:** Include a clear description in the PR outlining the changes, their purpose, and any additional information needed for approval.

- **Respond to Comments:** Address all PR comments within the PR. Focus on improving code quality and resolving issues highlighted, not on the number of comments.

- **Draft PRs and Discussions:** Consider raising a draft PR or having a small team discussion about the approach to avoid rework later.

- **Positive Tone:** Use polite language such as "Can you please" in comments. If you encounter a negative tone, interpret it positively and provide constructive feedback to the person or manager.

- **Approve Others’ PRs:** Ensure you approve pending PRs before requesting approval for your own.

- **Direct Responses:** Respond to PR comments directly within the PR. If there are delays, notify the reviewer that you have responded to their comments.

## Mentorship


As a team lead, one of your key responsibilities is to guide and support all team members, especially juniors. Here’s how I approach it:

- **Bi-Weekly Connects:** Schedule bi-weekly sessions with each team member.
  - Discuss any concerns they have.
  - Explore their career goals and interests.
  - Talk about any technical issues or topics.
  - Offer suggestions for their improvement.

I keep these sessions informal to better understand team dynamics, identify areas for improvement, and discuss potential work assignments and career paths.

## Sharing Credit/Recognition

As a team lead, it’s crucial to acknowledge and credit your team members for their contributions. Here’s how to effectively share credit and recognize efforts:

- **Give Credit:** Publicly mention the team member who did the work. Avoid taking undue credit for their efforts.

- **Recognize Efforts:** Keep track of contributions that deserve acknowledgment.

- **Recognition Methods:**
  - **Feedback:** Share positive feedback with the team member’s supervisor.
  - **Awards/Points:** Recognize their efforts through internal awards or thank-you portals.
  - **Thank You Email:** Send a detailed thank you email, cc’ing their supervisor or manager.
  - **Public Recognition:** Mention their contributions in meetings, presentations, or slides.

- **Be Specific:** Avoid vague recognition. Provide detailed feedback to show genuine appreciation and attention to their efforts.

**Bad Recognition Examples:**
- "Thanks for all your efforts last month."
- "Thanks for all your efforts in delivering project X."

**Good Recognition Examples:**
- "Thank you for your exceptional work on the X project. Your innovative solution to Y issue significantly improved our workflow. Your attention to detail and proactive approach were instrumental in meeting our deadline. Great job!"

{% include donate.html %}
{% include advertisement.html %}

## Listen To Feedback

It's important to listen to feedback as a Team Lead.

More often, Team Leads fail to gather feedback from their Team Members due to following reasons -

[1] They think they are doing a good job

[2] Lack of time

[3] Confusing the need for feedback with retrospective sessions

Below is a sample feedback you could get from your team every quarter, to know the team pulse - 

[1] Make sure every team member completes the feedback form

[2] Remove ambiguity in the questions

[3] Ensure your questions cover aspects you think are important for the growth, culture of the team

[4] Ensure you ask questions that aren't covered in retrospectives but are in your team member's mind which they don't ask due to some hesitation

<iframe width="640px" height="480px" src="https://forms.office.com/Pages/ResponsePage.aspx?id=DQSIkWdsW0yxEjajBLZtrQAAAAAAAAAAAAN__qYx7d9UQzRYTDI5NUQ0WEM5REZHRE5RSjQ4WEUyMC4u&embed=true" frameborder="0" marginwidth="0" marginheight="0" style="border: none; max-width:100%; max-height:100vh" allowfullscreen webkitallowfullscreen mozallowfullscreen msallowfullscreen> </iframe>

First step is to ask for feedback but even more important step is to act upon the feedback!

Most often, feedbacks asked by companies are not acted upon - empty promises are given such as 'we'll look into them', 'your feedback is really important to us' etc. after which the conversation dies and you end up filling similar feedback for next year.

Once you get the feedback do the following -

[1] See what suggestions are within your control and makes sense that you can quickly implement in the team

[2] Discuss with your manager on any items that need his guidance/approval/discussion with higher management

[3] Keep the team posted on a recurring basis on what you are doing to address the feedback items in your capacity

{% include donate.html %}
{% include advertisement.html %}

## Monthly Learning Sessions

Knowledge sharing is an important aspect to help the team grow, avoid silos, dependencies with an individual team member, career growth etc.

Each and every team member needs to be aware of how certain component works, not just from a code aspect but also from an hands-on approach as well as be given the opportunity to raise questions.

Although we could say that PR's are the platform for achieving some of the above, it is not always possible as sometimes certain members don't review PR's or give quick approvals due to lack of time/delivery constraints.

<br/>
Schedule a recurring Monthly Learning Session following below practices -

[1] Keep the learning session to a max of 1 hour - anything more than that will just make people to lose focus

[2] Make sure every team member contributes to the session - Have a form created before the session and ask each team member to fill it out in terms of what they would be sharing within the team - it could be something that they worked in the current sprint, any technology news/trends etc.

It shouldn't be the case where one team member keeps sharing the knowledge while others just keep listening. It's not a training class.

Each team member should bring something to the table.

[3] Time-box the session to 15 mins for each team member to present.

[4] Record the sessions and have them embedded in your internal documentation site so that external teams/new team members can simply listen to them instead of having to repeat same information every now and then.

[5] If your team has more team members than you could accommodate within a 1 hour session then split the session into 2 learning sessions on different day.

<br/>
*What does above achieve?*

[1] Work could be distributed instead of depending on same team member every time

[2] Anyone in the team could handle support/issues 

## Newsletters/Announcements

- Share what the team has delivered every sprint to stakeholders
- Share any new docs published for everyone's awareness
- Share proposed work items so that external/dependent teams can raise any risks/impacts

<figure>
    <a href="{{ site.url }}/assets/img/2022/05/newsletters.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2022/05/newsletters.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2022/05/newsletters.png">
            <img src="{{ site.url }}/assets/img/2022/05/newsletters.png" alt="">
        </picture>
    </a>
</figure>

## External Support

- Document any new questions that arise so that they could search and find the information themselves the next time or you could share link to the doc
- Assess priorities of support vs delivery


## Meetings

One of the problem that comes up in a Retro is `too many meetings` and how it impacts delivery.

Strategy that I have used is to represent the team in most meetings and invite the team members only when necessary.

This keeps the number of meetings that team members need to attend to a minimal.

As a team lead, you then collate all updates and share it in the standup/team's channel announcement so that everyone is kept in loop as well.


{% include donate.html %}
{% include advertisement.html %}


## Manager Discussions

- Discuss your goals/expectations
- Get feedback on your work
- Highlight any retro items
- Get inputs on any process you would like to try
- Use the expertise of the manager to see how you can solve any team problems
- Provide both strengths and weaknesses of team members on a recurring basis

## Globally Distributed Team

When you have a globally distributed team, some of the challenges which arise are:
- Less time overlap where all team members are available
- Team members blocked and waiting for info from members in another region
- Missing any meetings that happen after-hours in a different time zone
- Giving same updates in different standups e.g. Product Owner in US while other teams are in different locations

What I did:
- Use the overlap time effectively e.g. learning sessions, retrospective
- Split the standups to suit different timezones but don't ask to repeat same updates from team members who end up having to attend multiple standups. Instead, summarize the updates wherever applicable.
- If team members are blocked, have some doc tasks that they could work on in the meantime while members in other location come online for support/clarification
- Share any meeting updates via email/capture a card and give updates in the next standup to keep everyone in loop
- Foresee any information/potential blockers in advance and have them resolved/share information via email/channel notification so that the other time zone members have clarity/unblocked once they come online early the next day

To zoom and view the content, check the image here - https://raw.githubusercontent.com/HarshadRanganathan/team-leader-templates/main/meeting-schedule/meeting-schedule.png

<figure>
    <a href="{{ site.url }}/assets/img/2022/05/meeting-schedule.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2022/05/meeting-schedule.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2022/05/meeting-schedule.png">
            <img src="{{ site.url }}/assets/img/2022/05/meeting-schedule.png" alt="">
        </picture>
    </a>
</figure>

## General Tips

- Try what works for your team
- Do a round table with all team members to garner their thoughts and reach a collective conclusion - avoid being authoritative
- Sometimes what the majority suggests isn't always the best solution - at those times enforce authority and guide the team in the right direction
- Improve things step by step - you can't change the culture overnight
- Place trust in the team members - avoid micro management (use only when necessary)
- Market your team's work
- Give credit where credit's due


{% include donate.html %}
{% include advertisement.html %}