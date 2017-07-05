---
series: UI Design Problems;1
title: The UX problem of complexity in user interfaces
tags: tag1, tag2
author: Lukas Oppermann
category: design
description: How user interfaces get overly complicated and how you as a UX/UI designer need to fix it.
preview: Complicated user interfaces are a problem, especially in big organizations, with huge teams and more business minded product owners. As a UX or UI designer it is your job to fight this problem.
---

> As a designer your job is to reduce complexity. If your solution is not dead simple, you are not done yet.

As humans we have always been trying to simplify our lives. Take heating for example: First we had to collect fire wood to heat our caves. We replaced this with oil and later hot water, that was brought to our houses via pipes. All we have to do is turn the heater on when we are cold. Now the next step is to make the heater smart, so that it can turn on by itself.

## Identifying user requirements
Before we can come up with a smart product we need to identifying what it needs to be able to do. Heating seems pretty straight forward: Turn the heater to the desired temperature when you are cold, turn it off when you are too warm. But if we think about it a little more, there are other things to consider.
- you want different temperatures in the morning, when you get up, during the day and at night, when you are sleeping
- you are probably home more during the weekends, so you need a different heating period during this time
- in the summer time you will want a much colder temperature than during winter
- you might stay up longer on some random days
- your kitchen needs to be much warmer than your bed room, because most people prefer a colder temperature when sleeping

To sum this up, we need to provide different temperatures in different rooms at different times. They might also differ throughout the year. Sounds complicated? It is, if you are in the business of automating it.

## How not to do it
One simple way of dealing with this complexity, is to not deal with it. If you are building an app to control smart heating devices, you can just let the user take care of the problems. This could be by just providing a digital version of the on / off control, which means the user would have to turn the heat on or off via the smartphone. This does not move us anywhere closer to a **smart** heating system. If you think about it, this does not make it any easier, because you still have to control your heating manually. Except now you need your smartphone to do so.

Yet, many companies are still building apps like this. Why? The main reason is a misunderstanding of what a **smart device** actually is. Contrary to the believe of many, the idea is **not** just to make everything controllable via smartphone.
Just because you are using your mobile phone to control your heating, does not make it any smarter.

> What makes a smart home smart, is the ability to decide what to do by itself.

Another wrong approach is to add configurations for everything. This means the user decides beforehand what the device should do in a given situation. One problem with this approach is, that the developer would need to think of every possible scenario, to allow for the needed configurations. Another issue is, that the user needs to decide on all heating scenarios when setting up her heating system. Imagine setting up your new heating system in July and having to predict what temperatures you want in November. This approach avoids problems for the product developers by make them the users problems, not so good.

![Smart Home app heating](./media/smarthome-tkom.jpg)
<caption>In this app the user has to set up the temperatures for every day of the week in every room in advance.</caption>

## A better approach
If letting the user do the work is not the right way, you might be wondering what is actually a good approach. Well, let the machine do the work. This also means, let the machine do the thinking. Machines are not bad at thinking, they just need a little nip here and there.

A good example of this is the Nest thermostat, a replacement for your normal thermostat. This is important because it makes it easier for you to use it, as it does not require a change in behavior. You can adjust the temperature where you always adjusted it. But the really great part about it is, that it constantly tracks your changes to learn your personal heating desire. After a couple days it can predict when you want your home to be warm and when you want it to be cold. If your heating needs change, for example because you started to do home office, nest will adjust within a couple days.

![Nest](./media/nest.jpg)
<caption>Without a need to change behavior, the user is instantly familiar with her "new" smart heating.</caption>

## Extending upon success
The approach above is very good, but there is still room for improvements. To be fair, I do not know how much of this technology might actually be integrated into the Nest already.

If you start out with a "thinking" device, you are capable of nearly everything. To improve the experience even more, we could not only track the temperature the user sets on a seasonal basis, but also depending on the weather. If the user now allows the device to access the wifi, the temperature can be set depending on the outside temperature and weather situation. This now moves the device more towards a connected device. If you think along those lines an open window or a  locked front door could for example signal the device to stop heating.

## A really smart home
In summary, a smart home is not a bunch of devices that are controlled via a smartphone, but a collection of devices that can think for themselves, take appropriate actions depending on the circumstances and reduce effort and complexity in a given task. It needs to adapt to the user and try to replace the old devices as seamlessly as possible. **Remember: If the solution is not dead simple, you are not done.**
