---
title: 'Taking the Pager is Taking the Wheel, Not the Title'
date: 2023-12-16
description: Framing On-Call for Engineering Happiness and Success
---

This post was partly inspired by Sean Kilgore's recent blog about [Common Errors
in the Paved Path
metaphor](https://logik.al/posts/common-errors-in-paved-path-metaphor/). Sean's
language mirrors a metaphor that surfaced within my team about our On-Call
Rotation. Sean contrasts the "paved path" an organization provides to its
developers with the "vehicles" each team puts on that path. My team talks about
being on-call in terms of "taking the wheel of the vehicle," instead of the more
common "being responsible for fixing anything that goes wrong." 

## Taking a Turn at the Wheel

Most of us have taken a road-trip with friends at some point in our lives. On
such trips the group usually shared the driving duties. Everyone took a turn at
the wheel.

Taking a turn at the wheel came with certain responsibilites:

1. Understanding and obeying the rules of the road.
1. Protecting the life and safety of the occupants and cargo.
1. Protecting the vehicle and its ability to operate.

It _might_ also have come with some privileges:

1. Control of the air conditioning.
1. Control of the music.
1. Permission to wake people, if necessary, to protect the occupants or vehicle.

It certainly did _not_ come with the right to:

1. Have the vehicle repainted, re-upholstered, or upgraded to "our standards."
1. Sell the vehicle or trade it for one we thought was "better."
1. Force everyone to take another mode of transportation because "I don't feel
   like driving right now."

Put another way, the driver does not become the owner of the car. They also do
not become the mechanic, the tow truck, or the electrician.

If the engine was overheating we expected the driver to lower speed, ask
everyone to open their windows, and crank up the heater. We did _not_ expected
them to pull over and fix the radiator or perform an oil change on the side of
the road.

When a tire was punctured, the driver was responsible for pulling over. They may
or may not have been the one who changed the actual tire -- but at that point,
they were not driving.

There was no expectation that the driver be (or become) an expert on engines or
drivetrains or the car's electrics. They were just expected to be awake,
present, and commited to our safe arrival a the destination.

## Being On-Call is Taking a Turn at the Wheel

Being the Driver of a car is a great metaphor for being on-call. And here we are
talking about being the person who "carries the pager." If something alerts, it
will be your phone that rings. You are responsible for:

1. Acknolwedging the page and investigating the alert.
1. Assessing the _impact_ of the alert on the safety of the system or the
   business's ability to operate.
1. Taking any action necessary to preserve the safety of the system and the
   business.

That last item is often misinterpreted. The responsibility to take action
implies the right to run commands on production systems. On-call can restart
servers, kill pods, or manipulate the database. But it also authorizes them to
"wake someone up." As the Driver, on-call might know how to change a tire -- and
that is great -- but if they can't, they are expected to pull over and ask for
help.

A driver who continues to drive on a "flat" tire is not doing anyone any favors.

## Who is Not On-Call?

There can be push-back to the idea that on-call is allowed to wake up other
people. This often comes in the form of the question,"Wait, doesn't that mean
I'm _always_ on call?"

To answer this, I defer to Charity Majors' popular [ON CALL SHOULDN’T SUCK: A
GUIDE FOR
MANAGERS](https://charity.wtf/2020/10/03/on-call-shouldnt-suck-a-guide-for-managers/):

> As for engineers who write code for 24×7 highly available services, it is a
> core part of their job is to support those services in production. (There are
> plenty of software jobs that do not involve building highly available
> services, for those who are offended by this.) Tossing it off to ops after
> tests pass is nothing but a thinly veiled form of engineering classism, and
> you can’t build high-performing systems by breaking up your feedback loops
> this way.

So the answer is: yes. Engineers are _always_ on-call for _their service_.

This can raise the question,"If that's the case, then why doesn't each team have
it's _own_ on-call? And why can't the alerts route _directly_ to them? Why have
a shared on-call at all?"

Let's apply that question to our metaphor: Would you accept a condition where
your "engine is overheating" light pings your mechanic instead of you? Would it
become your mechanic's job to phone you or chase your car around to get you to
slow down or stop? That sounds like a lot of engines are going to be driven into
the ground by people who were never told there was an issue.

The fact is, many alerts can be mitigated without in-depth knowledge of the
application. That is the purpose of runbooks: to provide instructions on
mitigating, not fixing, problems.

I do support the idea of individual teams having an on-call rotation for
escalations. If the org is large enough, and has enough resources, then everyone
"can take their own car," and have their own alerts and rotations. But many
engineering orgs simply do not have the resources to support this. The idea I
want to put forward is that org-wide on-call rotations can succeed if management
clearly frames the goal of on-call when engineers join the rotation.

## The Goal of On-Call

I've heard that one motivation for org-wide on-call is "so everyone, eventually,
learns the whole application." This framing can lead to conflict and
frustration, as members of one team helpfully offer constructive criticism to
members of another in the name of "making on-call suck less." It all but
guarantees that people step on one another's toes. It fosters an "us vs them"
mentality, and usually makes on-call (and the whole org) suck more.

The goal of org-wide on-call is to maintain global awareness of how the app
behaves from the perspective of the customer. Even Charity Majors recommends
that managers put themselves in the on-call rotation. This is not to make them
full-stack engineers. This is because managers, also, need to be thinking about
the customer impact of changes moving through the pipeline.

We build applications to serve customers. That's the difference between being in
business and being in a hackathon. No engineer is made better at their job by
ignoring the metrics that impact the customer's experience. No code is better
for being less observable, no architecture is better for being impossible to
troubleshoot (or impossible to manipulate at runtime).

It is not reasonable, and it is not safe, to constantly treat the holistic
health of the app as "someone else's job." It is not reasonable, and it is not
safe to force a small group of people do all the driving. In the name of safety,
and fostering common purpose, many orgs opt for the org-wide rotation, with
everyone taking a turn at the wheel.

## Supporting On-Call

Of course, this re-framing is only half the job. If Engineers are going to be
"driving" during on-call week, that means they're not going to be doing anything
else. Going back to our metaphor, imagine being asked to take your turn at the
wheel, and then the person handing you the keys says:

"Oh -- and since you're driving, you should also be able to tune the carburetor
and rotate the tires. Because you're driving anyway."

Doesn't make a lot of sense, does it?

Loading up on-call engineers with sprint tickets forces them to switch context
when an alert comes in. This can cost them, and the org, up to
[40%](https://www.psychologytoday.com/us/blog/brain-wise/201209/the-true-cost-of-multi-tasking)
of their productivity. 

If these engineer cares about their performance reviews, that could mean they
put in 40% _more_ time and effort to catch up. This leads to frustration,
conflict, and burnout. On the other end of the spectrum, they could truly divide
their efforts and attention. With all the context switching, that means their
remaining 60% of effort is split between on-call and sprint work. That's 30%
effort on either task.

Giving on-call engineers sprint work is asking for less than 1/3 of an engineer
on that sprint work, and less than 1/3 of an engineer on-call.

That is not reasonable, and it's not safe.

Framing On-Call in terms of Engineers "Taking a Turn at the Wheel" helps both
Management and Engineering maintain a clear understanding of the boundaries that
make these org-wide rotations successful.