Title: Google Meet Bot attends online lessons for me

## Before Reading
As always, the source code for this Google Meet Bot is available on my <a href="https://github.com/EmaMaker/GoogleMeetBot">github</a> profile. You can even download it and set it up for your own use following the instructions! If you want to see the bot in action take a look here

# Video Demo <a href="https://www.youtube.com/watch?v=ZaBPLZkqqas">(YouTube)</a><a href="https://invidious.snopyta.org/watch?v=ZaBPLZkqqas">(Invidious)</a> 


## The Problem
I’ll be honest, I’ve never been the kind of person that wakes up early in the morning and it’s super energetic right off the bat. Even if i wake up at 7am, getting out of the bed takes me ages. Now, since the COVID-19 outbreak in my country, schools have moved to online lessons countrywide. Even if by now we go physically to school for 75% of the week, we still have to attend online classes for the other 25%. This whole time, knowing I had to stay at home to follow my lessons made waking up early even harder for me. And apparently keeping note of who arrives to a lesson late is apparently a huge problem, which neither me nor my teachers want to deal with. 

## What do do?
I had to do something for this situation, and getting up in time was simply not an option. I needed something that could join lessons on time for me, which allowed me to sleep more. So i needed something to automate my computer, but how? I could write some program that simulated the mouse moving and clicking, but I had to do it in the past and I knew it was terrible idea, so I ditched it. Yet, i needed something to simulate a person using the browser.

## Looking at old projects

as I said, using some kind of program to virtually move the mouse and click stuff was totally not a good idea. At least not writing my own. Then i remembered that some time ago I worked on an <a href="https://github.com/EmaMaker/AccountChecker">Account Checker script</a> for a friend. It was basically a program which, given a list of usernames and passwords, would try to login into websites. This was a nice and quick project that you can find here. I might do a post on it some day.

The Account Checker was based on the <a href="https://github.com/nsgodshall/Hatch">nsgodshall’s fork</a> of <a href="https://github.com/zshell/Hatch">zshell’s Hatch</a>, it was written in Python and used a module called  <a href="https://www.selenium.dev/">Selenium</a>, which was exactly what i needed.

## Selenium
It’s a simple yet extremely powerful website automation tool. Selenium can open an automated Firefox window, and it can simulate the click of buttons or typing of keys like if a person was using it.
To do this, the browser has to be ready for automation. For Chrome, this means installing the chromedriver and giving the right path to it to the script. For Firefox, it means doing the same thing with the geckodriver.
I decided to settle with Firefox because Chrome was giving me some strange problems with login in the early tests. While I was thinking of this, and had already made a somewhat working example, a classmate of mine, to which I had spoken about this project before, sent me <a href="https://www.youtube.com/watch?v=7neSueHsyY0">this video by TechRaj</a>. TechRaj made a bot to automatically join Microsoft Teams meetings, the same concept could be applied to Google Meet, but it had some serious problems to be solved.

## The Bot

The most important problem in this case, is that Google, unlike Microsoft, doesn’t allow google accounts to be joined from automated browser sessions. Second problem, is that, unlike TechRaj does, CSS Selectors cannot be used to select the components, since the selector of a component changes on a per-meeting basis on GMeets. This second problem is quite easy to resolve by using XPATHs instead of CSS Selectors. An XPATH just a string that describes the position of a component in an HTML structure. XPATH are static, unless someone changes the HTML structure of the page, which shouldn’t happen really often on a service like GMeet.

The account problem is a little trickier to workaround, but I managed to find a way. FireFox has profiles, which retains information about the browsing history, cookies and, most importantly, accounts that are logged in. So we could create a new FireFox profile, manually login into the needed Google Account via GMail and let the profile store the account. Since this step has been done manually, without an automated browser, Google will let it login into the google account with no problems.

Then, using Selenium, we can open a new FireFox window that uses the profile we just created, and it will be automatically logged in into Google, struggle free.

from this point on is just a matter of using **WebDriver.get**, **WebDriver.click** and **WebDriver.send_keys** to click redirect the browser to the given Meet link, click the buttons to mute camera and microphone, join the call and eventually type something in the chat. At this point there’s only a scheduling system missing.Scheduling

It wouldn’t be a real bot if it wasn’t able to automatically join classes at the right time, every day. Searching around i found the **schedule** module for python. Basically, as long as script runs, it will execute the given action at the right time of the right day. I did another bit of manual work and programmed the bot with my current lessons schedule.

Of course, a teacher could be late, so the bot is programmed to retry joining a meeting every minute if a failure of any type should occur.

Running in the background is the only thing left: my computer runs ArchLinux with KDE Plasma as Desktop Environment. KDE Plasma allows the user to create multiple Activities, which are basically virtual desktop isolated from each other. I created an activity for my school stuff, and let the bot run there.

## Conclusion and Final Considerations

The bot is lightweight and easy to set up and use. I’ve been running this Google Meet bot for the past couples of months and hasn’t failed on me since, and I still haven’t got busted by my teachers.

# Video Demo <a href="https://www.youtube.com/watch?v=ZaBPLZkqqas">(YouTube)</a><a href="https://invidious.snopyta.org/watch?v=ZaBPLZkqqas">(Invidious)</a> 
