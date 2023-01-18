Title: MakerFaire Rome 2017

# MakerFaire Rome 2017

## MakerFaire 2017 and our project

The December of 2017 started with a beautiful experience: I took part, with some other friends and our parents, to the MakerFaire 2017. We had our own stand and we showed to the big public of the Faire our project. What was is about? It's called 3DPacRobotMan and it's a real-life version of the Pac-Man game. Pac-Man is a videogame that came out in the '80s and still today is very famous.

<img src="//emamaker.com/resources/events/makerfaire2017/makerfaire1.jpg" />
Our version uses new technologies to make a funnier and cooperative version of the game:

The field is all made of laser-cut wood. On the underside, the field is divided into sections. Every section has a board that controls a group of reed sensors. This board communicates in SPI with a central shield, which takes count of the reed sensors activated under all the field, turning off the corresponding LEDs, mimicking the coins in the original game.
As in the original game, there are five characters: one is the Pac-Man and the others are the ghosts. The ghosts have to catch the Pac-Man, which has to escape from them. If the ghosts catch the Pac-Man they win, otherwise is the Pac-Man to win. Again, a reed sensor on the PacMan recognizes when the ghosts touch it. The robots are made using Arduino Uno modified Boards and remote controlled with Radio Frequences @2.4GHz.

I took care of the scenic (so to speak) part: a Raspberry Pi serial communicates with the field to let the players see the score, the highscore, and plays the game sounds.

<img src="//emamaker.com/resources/events/makerfaire2017/makerfaire2.jpg" />
 
_Note as of 05 Jan 2023, as I'm porting this to the new website: Everything was made in Java Swing. This was the only language I knew at the time, and I admit it was pretty bad. Swing GUIs do not update if no callbacks are called, activated by user action. My workaround was to create a button with transparent background and no text on it, and I had a script moving the mouse and simulating left clicks (I do not remember if it was external, launching a script from the terminal or integrated in my Java program).
If I was to do it again today, I would make it in Processing: easy serial communication with arduino, no reason to do weird workarounds to update the gui, easy sound playing_
_Cherry on top, I actually lost the source code to this master piece of a program._

<img src="//emamaker.com/resources/events/makerfaire2017/makerfaire3.jpg" />
## 3d Pac Robot Man on 'The MagPi'
Do you know the Raspberry Pi? It's a credit-card sized computer, that you can buy for just 35$.
The Raspberry Pi Foundation has its official magazine, called "The MagPi". I had the luck (and the guts to ask them) to speak with the Raspberry Pi Foundation guys at the MakerFaire 2017 and I told them about our project.
They liked it and they published and article about our project in the February 2018 Issue of the MagPi. Down here there are the photos of the two pages about our project, but you can always download it for free from <a href="https://emamaker.com/magpi.raspberrypi.com/issues/66">The MagPi's official website</a>. I also have a <a href="https://emamaker.com/emamaker.com/resources/events/makerfaire2017/TheMagPi66.pdf">mirror on this website</a>, should the link be dead.
<img src="//emamaker.com/resources/events/makerfaire2017/makerfaire4.jpg" />
<img src="//emamaker.com/resources/events/makerfaire2017/makerfaire5.jpg" />
