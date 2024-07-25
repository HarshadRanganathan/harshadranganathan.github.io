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

We would like team members to contribute docs on a recurring basis - when they find time, working on related tasks etc.

But in reality this doesn't happen as team members are busy with delivery.

So, we have to start enforcing things.

One idea I had implemented is to ask team members to contribute docs on a weekly basis.

We chose `Friday` as the day team members will spend an hour or so for writing docs. Friday is usually when team members wind up on items, bit relaxing day so it made sense to utilize that day and ask members to contribute for docs.

Next is how to enforce it -

1. We added a reminder card to Teams planner tab - e.g. Friday Docs with reminder/due date
2. On Thursday standup, we will bring up the docs backlog list and pick items we think are of quick wins/priority to be documented
3. As a team, we would discuss 

    what each member will be contributing on and assign items<br/>
    check if sufficient details are available for the task<br/>
    clarify any doubts

4. On Friday, team members would spend some time to contribute docs and raise PR

On a recurring basis, this practice worked well.

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
            <img src="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png" alt="">
        </picture>
    </a>
</figure>

Problems faced:
1. Requires the standup facilitator to bring up the card on Thursday's
2. Sometimes, team members still end up not delivering on docs so need to remind for next week

{% include donate.html %}
{% include advertisement.html %}

### External Contributors

If you have limited team members/more items to document, it might make sense to bring in external contributors to scale up the activity.

Couple of approaches to bring in external contributors:
- Market that they get a chance to work and learn with your team on the topic they document
- Incentivize external contributions by giving thank you points, thank you emails or callouts in meetings.
- Don't fix hard deadlines for doc delivery

Although this involves spending little bit more time with external contributors on explaining the process, walkthroughs, supporting them by providing required details etc.

### PRs

- If PR's don't have README updated (depends on the change) suggest to have it done before giving approval

- If PR's need to be approved due to urgency of the change, then ask the team member to raise an issue in the repo so that it's tracked as a pending item to be worked upon later

{% include donate.html %}
{% include advertisement.html %}
## Daily Standup

- Assign standup facilitator or rotate it among team members every week/sprint
- Ensure quick round table of updates 
- If any team member derails the standup updates remind them to instead bring it up in the parking lot as every team member needs to first finish providing their updates
- Ensure standup doesn't exceed the alloted time limit
- If any items require more discussion check with the team to see if they wish to continue in the same meeting/provide small break/ask the team member to create a dedicated meeting for further discussion etc.

Also, as a team lead the daily standup is a good place to provide:
- any general updates 
- check status on any specific work items
- any specific items to be briefly discussed

### Standup Cards

Sometimes, people forget what they had planned to raise in the standup.

So, I introduced a process to raise cards in the teams planner tab called `Daily Standup Items` where team members/team lead can quickly add a card with due date for discussion as and when it comes to their mind.

Then, in the daily standup ceremony we go through these cards and the member who raised it provides updates/asks questions. In this way, we don't miss things to be raised in the standup.

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png">
            <img src="{{ site.url }}/assets/img/2021/12/teams-planner-card-friday-docs.png" alt="">
        </picture>
    </a>
</figure>

Problems faced:
1. Needs to be part of regular ceremony of things done in standup
2. Sometimes, there might be more cards so try to pick cards which are important from team's perspective to be discussed right now and cards which can be delayed
3. Some team members prefer to have their own personal notes

{% include donate.html %}
{% include advertisement.html %}

## Retrospectives

- In a tool of your choice, have different columns such as `Start`, `Stop`, `Continue` and `Action Items`
- Rotate the retro facilitator every sprint
- Start timer for `10-15 mins` and have the team members populate the cards for each section
- Retro facilitator goes through the cards and works with team to see what cards can be merged (as duplicates) and what cards need more clarity
- Start timer for `5 mins` to vote on cards - each team member is given certain number of votes which they use to indicate what is important to be addressed
- Retro facilitator works with the team to generate action items and ownership for the cards

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/retro-board.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/retro-board.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/retro-board.png">
            <img src="{{ site.url }}/assets/img/2021/12/retro-board.png" alt="">
        </picture>
    </a>
</figure>

Typically, every team struggles with retrospective action items:
- card is not actionable since it requires more authority
- team lead/member who has the action item assigned is busy so they come up with an excuse of `I was busy/no time/forgot` in the next retrospective
- not all cards have action items assigned - so a team member could feel that their card is ignored

In the end, retrospective becomes more of a place where people can vent their frustrations on a recurring basis and less problems get resolved.

What I did:
- Even if a retro card didn't get the votes if it feels to be an important item to be addressed even for a single team member then spend some time to provide tips/address concerns/generate an action item for it
- Pick only reasonable action items - things that could be solved by the team/by manager - anything that requires more authority raise it as a feedback to manager 
- Add a card to `Daily Standup Items` with a due date set to mid of the sprint to have a `Retro checkpoint` to share any updates on the action items/to remind team members of pending action items etc.

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

- Do frequent groomings instead of one big grooming call e.g. have the grooming meeting every week or break it down to smaller sessions based on team feedback
- Encourage team conversations rather than you just providing all the required details
- Remove any fear within team members by having conversations with them that it is okay to say you don't understand the story or it needs more details
- Do a round table if you notice that any team members are not participating by providing their inputs/raising questions/feedback
- Ensure that the story has enough details for anyone in the team to work on it
- Use any tools to get the story point size from all team members 
- Always try to go with a bigger estimation when in doubt rather than a smaller one
- If any team member gives very high estimation ask them the reason to see if it's justified - split stories to multiple smaller ones if needed
- For any stories that are new to the team check if a spike is needed - if so try to have it in the sprint prior to the work so that feasibility/approach to be used/gotchas are known prior to the implementation. Based on the spike further discussions can happen
- If you have new team members they may not be in a position to know if the story has required details - ask them to go through the story during the week and come up with any questions for the next grooming call

Problems faced:
- Getting team participation is always a difficult task - it requires commitment from every team member
- Some team members end up asking more questions/indicate they have no idea on what needs to be done after picking up the story
- Sometimes team members assume that they may not be working on a particular story so they don't need to worry about if adequate details are present in the story
- More experience/domain knowledge may be needed to assess impacts/risks associated with a work item

{% include donate.html %}
{% include advertisement.html %}

## PI Planning

- Use any whiteboarding tools to plan work items across sprints
- Get inputs from team members to allocate any technical debt/backlog items that they would like to focus on in the upcoming PI and see if it can be accommodated
- Add sections/different areas that the team could focus on - e.g. CI/CD, security etc. so that everything is covered
- Prioritize the backlog to see potential items that could be worked upon to fill any gaps in PI
- Publish the planning to other dependent teams to that they could take a look at raise any items missed out

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

Ideal state:
- Team members pick tasks to work on as they finish each item
- Team members indicate if they are blocked/need additional support
- Team members are aware of the priority tasks

Reality:
- Team members wait for tasks to be assigned
- Team members take additional time before they reach a conclusion they they need more support/mention it only when asked
- Team members wait for standup to raise that they are blocked - instead they could have quickly mentioned in group chat
- Team members work on low priority items/not aware of priority

What I did:
- To go to the ideal target state, we need to go step by step - so started assingning tasks explicitly to team members while working to develop a culture where members pick tasks themselves based on priorities
- If there is a sense that a team member seems to be struggling or taking more time to deliver, check with them if they need more support or swap tasks as required
- Sort the work items in the sprint board based on priority so team members always pick from the top
- Incase, the priority keeps changing - have a weekly Monday call to clarify the priority items for the week

Problems faced:
- Team members don't want to work on a particular task - boring/trivial/other reasons - but at the end of the day we are paid to do it and someone has to work on it
- Multiple team members interested to take up same work item - determine who to assign based on various factors such as priorities, previous work assignments etc.

{% include donate.html %}
{% include advertisement.html %}
## Design Discussions/Spikes

- Sometimes team members jump on a hype train/trending tech/pick something they learnt in a course and decide to use that approach for a story
- Always as part of a spike, ask the team member to come up with the list of approaches they evaluated, pros/cons of each approach, recommendations and schedule a team meeting within an agreed time box
- Share analysis report with the team in advance before the meeting so that they can read through it and come prepared
- In the team meeting, evaluate the approaches and recommendation from the team member, ask questions and decide if further analysis needed based on new inputs/risks/impacts
- Prefer incremental improvements
- Assess if the solution is unnecessarily complicated/over-engineered

<figure>
    <a href="{{ site.url }}/assets/img/2021/12/spike-analysis-template.png">
        <picture>
            <source type="image/webp" srcset="{{ site.url }}/assets/img/2021/12/spike-analysis-template.webp">
            <source type="image/png" srcset="{{ site.url }}/assets/img/2021/12/spike-analysis-template.png">
            <img src="{{ site.url }}/assets/img/2021/12/spike-analysis-templates.png" alt="">
        </picture>
    </a>
</figure>

Problems faced:
- Theoretical vs Practical analysis
- Delivery constraints force options

{% include donate.html %}
{% include advertisement.html %}

## Technical Backlog

- Prioritise items in the backlog so that you can quickly pick any items/debt the team is interested o work upon to fill any gaps in sprint
- Work with manager/product owner to have a buffer in the sprint to accommodate tech debt items


## JADs

JADs (Joint Application Design sessions) help to flush out solutions to problems by involving all stakeholders.

- Set prior agenda and send it in advance
- Keep the size of the group small otherwise you will have difficulty reaching conclusions
- Time box to avoid lengthy discussions leading to less meaningful outcomes
- Capture meeting notes, decision points and have them stored in code repository/docs for future reference
- Interrupt politely to have any topics to be taken outside the meeting which is not in scope
- Sometimes it's better to come with recommendations rather than expecting it from the meeting
- Discuss with team members to ensure every one are on the same page - last thing you want is your team member pitching something different in the call causing more chaos
- Try to influence the meeting in the direction you want to take


## Priorities

- Indicate priorities clearly to the team
- Order the sprint items based on priorities so that team members pick from the top and work towards the botom
- Check if team members are struggling with any priority items and get them the required support to avoid delivery delays
- Indicate to the team as and when priorities change


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

Typical problems:

- PR's not reviewed in a timely manner
- PR changes are too large
- PR review comments ignored
- Too much changes proposed
- Rework of PR needed
- Tone of the person's comment
- Team Member doesn't approve other's PR but expects his PR to be approved
- PR sits for more than a week
- Change suggested in one place is not reflected wherever applicable
- Instead of responding to PR comments, you receive pings with explanation

Things I suggest to team members every now and then:
- Dedicate time every day to review pending PR's
- If you're waiting for a PR to be approved, post reminders in team's channel or raise it in the standup if it's a blocker
- Break PR's to smaller reviewable chunks - don't mix things up - do iterative improvements
- Provide description to the PR stating the change, purpose, any other info required for approval
- Provide your response to all PR comments
- Do not count how many comments are given - focus on improving the code, making it more maintainable and resolving any issues highlighted
- Raise draft PR or have a small team discussion on the approach you are planning to take to avoid any re-work later
- Use keywords such as 'Can you please' or suggestions in your PR comments. If you are reading the comments ignore the tone of the person and take it with a positive thought, share feedback with the person/manager to add it as a area of improvement for that person.
- Approve any pending PR's before demanding your PR to be approved
- Respond to PR comments in the PR itself - incase of any delays then reach out to the person to notify that you have responded to their comments

## Mentorship

One of the responsibilities of a team lead is to guide junior/all team members.

Strategy I had used is to have bi-weekly connect sessions with all team members.

- Discuss any concerns team members have
- Discuss their goals
- Discuss things they are interested to work on
- Any technical discussions
- Suggestions for improvement

I try to keep these sessions informal and it has helped a lot to understand conflicts happening within the team, areas for improvement, determine potential work assignments, career paths etc.

## Sharing Credit/Recognition

As a Team Lead, always share credit to the work done by your Team Member. e.g. mention that person X worked on it in a call instead of projecting that it was your efforts and getting undue credit

Recognize the efforts of your Team Members.

Take notes of your Team Members contributions that you feel needs to be acknowledged.

On a recurring basis, recognize the efforts by -

[1] Sharing feedback with the Team Member's supervisor

[2] Recognizing the efforts through any internal awards/points/thank-you portal

[3] Sending `Thank You` email cc-ing the Supervisor/Manager

[4] Calling out contributions in any meetings/slides etc.

Be more concrete with your recognition. I have seen many people who send thank you with vague content.

Capture every detail as possible in your email/reward so that the Team Member gets astonished by the level of observation by the Tech Lead.

**Bad Recognition Examples**

*Thanks for all your efforts last month*

*Thanks for all your efforts in delivering the project X*

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