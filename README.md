# TouchDesigner Project - Lighting and Video Control
Lighting and Video control through TouchDesigner. Implrementation of class based approach for some of the lighting commands with Python

![gui](https://i.imgur.com/XbfV9E6.png)

The following code is a class called lightingControls which has different methods which allow for different parameters controls of overall level as well as individual lighting channels.
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

## Scripts for running cue, reset and black out buttons
![scripts](https://i.imgur.com/gMfwKTS.png)


```
op('lighting').Cue()
op('lighting').ResetScenes().FadeIn(0.5).FadeOut(1)
```

The blackout button is also a CHOP Execute button which running the following code below. When active, the button turns red to give visual feedback to the user to know "blackout" is active. Once turned off, the color returns back to normal and the lights are set to full by increasing the overall intensity/master fader level of the output to the DMX OUT CHOP.

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
  
  d
