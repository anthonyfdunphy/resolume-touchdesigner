# TouchDesigner Project - Lighting and Video Control

<img alt="header_image" width="100%" src="https://i.imgur.com/hU8NK1C.jpg" />

Detailed here is my project, which involves the creation of a control system using TouchDesigner to manage the lighting and video scenes of a show. The project aimed to streamline the control of video scenes called "stories" that were comprised of video sections accompanied by audio playing on a 360-degree speaker. The system that I developed in TouchDesigner enabled seamless control of the lighting using DMX output, and also allowed for the sending of OSC commands to Resolume to manage a multi-projector setup. With this system in place, the show's operation was simplified, and all that was needed was to hit the "cue" button, triggering the logic that I created in TouchDesigner to run the show smoothly. In order to further enhance the functionality of the control system, I implemented a class-based approach with Python for some of the lighting commands. This approach helped to improve the organization and efficiency of the system, making it easier to manage and modify as needed.

### Control GUI
<p float="left">
  <img src="https://i.imgur.com/XbfV9E6.png" width="480" />
  <img src="https://i.imgur.com/KJju3kT.png" width="500" /> 
</p>

The class contains a variety of different methods that can be used to control the overall level of the lighting, as well as individual lighting channels. This approach is especially useful when it comes to making changes to the system, as it allows for modifications to be made in a more streamlined and efficient manner.

By abstracting away the complexity of the individual CHOP operators and instead focusing on the higher-level controls of the lighting system, the class-based approach that I developed makes it easier to modify the lighting setup in a more holistic manner. This approach can save a significant amount of time and effort, as it reduces the need to manually modify each individual operator. I found it to be most usefull for setting fade times and overall intensity of the DMX output.

<p align="center">
  <img alt="scripts" width="50%" src="https://i.imgur.com/Jz1DTgm.gif" />
</p>

<details>
  <summary>Expand code</summary>
  

  ### Lighting Control Component
  ```
"""
Extension classes enhance TouchDesigner components with python. An
extension is accessed via ext.ExtensionClassName from any operator
within the extended component. If the extension is promoted via its
Promote Extension parameter, all its attributes with capitalized names
can be accessed externally, e.g. op('yourComp').PromotedFunction().

Help: search "Extensions" in wiki
"""


class lightingControls:
	"""
	lightingControls description
	"""
	def __init__(self, ownerComp):
		# The component to which this extension is attached

		self.ownerComp = ownerComp
		
		#initalising fade times
		op('attributes').par.value0 = 0
		op('attributes').par.value1 = 0
		
		op('dimensions').par.value0 = 0
		op('dimensions').par.value1 = 0

		#default master set to blackout
		#op('lighting_master').par.value0 = 0

	def ActiveScene(self):
		return str(op('active_scene')[0,0])

	def SetDimensions(self,w,h):
		op('dimensions').par.value1 = w
		op('dimensions').par.value0 = h
		return self

	def Fade(self, time):
		op('attributes').par.value0 = time
		op('attributes').par.value1 = time
		return self

	def FadeIn(self,time):
		#time is seconds
		op('attributes').par.value0 = time
		return self

	def FadeOut(self,time):
		#time is seconds
		op('attributes').par.value1 = time
		return self

	def Cue(self):
		#this conditional was added to prevent stepping out of index range when cueing scenes
		if op('index').par.value0 == len(op('SCENE_SWITCHER').inputs) - 1:
			op('index').par.value0 = op('index').par.value0
		else:
			op('index').par.value0 = op('index').par.value0 + 1
		return

	def ResetScenes(self):
		op('index').par.value0 = 0
		return self

	#sets the intensity slider to 0
	def SetIntensity(self, i):
		op('lighting_master').par.value0 = i
		return self

	#sets the intensity slider to 1
	def Full(self):
		op('lighting_master').par.value0 = 1
		return self

	#sets the intensity slider to 0
	def Blackout(self):
		op('lighting_master').par.value0 = 0
		return self

	#set individual dimmer channel values
	def SetDimmerOn(self,chanNum):
		channels = op('all_channels')
		#the -1 is added to start at the intented channel
		setattr(channels.par, 'value{}'.format(chanNum-1), 1)
		return self

	def SetDimmerOff(self, chanNum):
		channels = op('all_channels')
		#the -1 is added to start at the intented channel
		setattr(channels.par, 'value{}'.format(chanNum-1), 0)
		return self

```
</details>

## Scripts for running cue, reset and black out buttons
<p align="center">
  <img alt="scripts" width="50%" src="https://i.imgur.com/gMfwKTS.png" />
</p>

```
op('lighting').Cue()
op('lighting').ResetScenes().FadeIn(0.5).FadeOut(1)
```

The blackout button is also a CHOP Execute button which running the following code below. When active, the button turns red to give visual feedback to the user to know "blackout" is active. Once turned off, the color returns back to normal and the lights are set to full by increasing the overall intensity/master fader level of the output to the DMX OUT CHOP.


### Blackout Code
  ```
def onOffToOn(channel, sampleIndex, val, prev):
	op('lighting').Blackout()
	return

def whileOn(channel, sampleIndex, val, prev):
	op('blackout/text').par.bgcolorr = 1
	return

def onOnToOff(channel, sampleIndex, val, prev):
	op('lighting').Full()
	return

def whileOff(channel, sampleIndex, val, prev):
	op('blackout/text').par.bgcolorr = 0.2
	return

def onValueChange(channel, sampleIndex, val, prev):
	return
```

<p align="center">
  <img alt="scripts" width="90%" src="https://i.imgur.com/04CN6WG.jpg" />
</p>

## Resolume Control
The different lighting scenes are changed through and index value on a SWITCH CHOP. The index value is tracked - if a scene with the name "story_" in it e.g. "Story_1" is active,the counter is added up for each story and then this selected the corresponding column in Resolume. If the index value is 0, the the video counter is reset back to 0 so it is in sync with the lighting scenes.

```
count = 0

def onValueChange(par, prev):
	global count
	
	index = par.eval()
	
	cueList = op('lighting/SCENE_SWITCHER').inputs[int(index)].name
	scene = "story_"
	
	if scene in cueList:
		op("video_on_off").par.value0 = 1
		count = count + 1
		#setting the video column in Resolume to match the index value of "story"
		columnSelect = '/composition/columns/{}/connect'.format(count)
		op('main_osc_out').sendOSC(columnSelect,[1])
	else:
		op("video_on_off").par.value0 = 0
	
	#reset video counter to 0
	if index == 0:
		count = 0
	
	return
  ```
  
## Video Effects
As part of the show, audience members were able to step on platforms in the performance area. These platforms were developed by a 3rd party team which were self containing sensor which outputed OSC values. As these platforms were going to control audio effects in Ableton, we create the show in a way so that the sensor levels that were read as a person entered onto the platform were passed from Ableton to Touchdesigner using the TDAbleton Component. As the person moved around the sensor, glitch effects in Resolume were increased in value.
<p align="center">
  <img alt="interaction" width="50%" src="https://i.imgur.com/wzeONSx.jpg" />
</p>

### Example Script
```
def onValueChange(channel, sampleIndex, val, prev):
	noise = '/composition/layers/3/video/effects/distortion/opacity'
	parent(2).op('main_osc_out').sendOSC(noise,[val])
	return
```
