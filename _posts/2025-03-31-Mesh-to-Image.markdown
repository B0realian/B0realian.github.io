---
layout: default
modal-id: 9
date: 2025-03-31
img: m2i.png
alt: m2i_altsm.png
project-date: March 2025
client: Nornware
category: Tooling
description: Creating terrain out of hi-res meshes
---
So, it was time to make the specified terrain creator. I repurposed some of the code from the previous attempt and decided to use the very excellent STB lib by Sean Barrett because there was no way I was going to spend all the time to write my own loaders for .png or .jpg and I really couldn't expect the user to convert all textures to .tga. I did, however, write my own loaders for .obj, .fbx (admittedly using the fbx sdk) and .gltf meshes. To parse the json "header" of .gltf I included the also excellent json.hpp by Niels Lohmann. Why? Partly because I knew I could and it would be a good learning experience and partly because I wanted the added portability: smaller file size and fewer dependencies. This was also something that low level wizard Johno of Nornware (where I was doing my internship) respected and encouraged.

Creating an array of vertices from mesh data is typically straight forward. However, I *did* groan when I realised that the gltf-format supported data embedded as a base64 encoded string. Incidentally, this is why you should only use .gltfs with embedded data if running webGL (or another web graphics API): load time and performance takes a definite hit from decoding the data. However daunting I initially found the prospect of writing a decoder, I was surprised when my first attempt just worked. Partly, this stemmed from not having copied someone else's decoder: I had to look up what base64 entailed which detailed some of the necessary steps (such as what characters are included and in what order) but otherwise I just wrote what I figured should be how it works.

```cpp
const char b64table[64] = { 'A','B','C','D','E','F','G','H','I','J','K','L','M','N','O','P','Q',
				'R','S','T','U','V','W','X','Y','Z','a','b','c','d','e','f','g','h',
				'i','j','k','l','m','n','o','p','q','r','s','t','u','v','w','x','y',
				'z','0','1','2','3','4','5','6','7','8','9','+','/' };

uint32_t start = static_cast<uint32_t>(uri.find("base64,") + 7);
uri = uri.substr(start);				// Assuming encoding will always be an octet stream.

int padding = uri.length() % 4;				// This should not be necessary but is a safeguard.
if (padding > 0)
{
	for (int i = 0; i < padding; i++)
		uri.append(padding, 'A');
}

for (int i = 0; i < uri.length(); i += 4)
{
	std::string tempString = uri.substr(i, 4);
	unsigned char uriSub[4];
	std::copy(tempString.begin(), tempString.end(), uriSub);
	unsigned char from64[4];

	for (int c = 0; c < 4; c++)
	{
		int idx = 0;
		for (char x : b64table)
		{
			if (uriSub[c] == x)
			{
				from64[c] = (char)idx;
				break;
			}
			idx++;
		}
	}

	unsigned char byteTriplet[3];
	byteTriplet[0] = (from64[0] << 2) + (from64[1] >> 4);
	byteTriplet[1] = (from64[1] << 4) + (from64[2] >> 2);
	byteTriplet[2] = (from64[2] << 6) + from64[3];
	binData.push_back(byteTriplet[0]);
	binData.push_back(byteTriplet[1]);
	binData.push_back(byteTriplet[2]);
}
```

So, I could load the types of meshes I wanted with the types of textures I wanted and stick them in front of the camera. But, rather fortuitously, I had accidentally clamped the camera to 90 degrees for the pitch which caused severe gimbal lock. The camera code was something I had got from a tutorial, and I have seen very slight variations of it across the web (searching for arcball camera). Now, that I was forced to revisit the code I immediately started to question it.

```cpp
void MoveCamera(float dYaw, float dPitch)
{
	dPitch = glm::clamp(dPitch, -89.f, 89.f);
	yawRad = glm::radians(dYaw);
	pitchRad = glm::radians(dPitch);

	camPosition.x = subjectPos.x + camRadius * cosf(pitchRad) * sinf(yawRad);
	camPosition.y = subjectPos.y + camRadius * sinf(pitchRad);
	camPosition.z = subjectPos.z + camRadius * cosf(pitchRad) * cosf(yawRad);

	model = glm::translate(model, subjectPos);
	view = glm::lookAt(camPosition, subjectPos, camUp);
	projection = glm::perspective(glm::radians(45.f), (float)mainWindowWidth /
						 	(float)mainWindowHeight, 0.1f, 100.f);
}
```

The code above converts degrees to radians before rotating the world around the camera. For this reason, the pitch needs to be clamped or the mesh suddenly flips as you pass cos 90 during camera movement. First off, I realised it is unnecessary to use degrees in the first place, and secondly I figured that if I rotate the mesh instead, there is no need to clamp the pitch, plus the math becomes much more intuitive.

```cpp
static void __camera_projection(glm::mat4& model, glm::mat4& view, glm::mat4& projection)
{
	model = glm::rotate(model, __state.subjectRotation.y * 1.5f, glm::vec3(0.f, 1.f, 0.f));
	const float xRot = cosf(__state.subjectRotation.y * 1.5f);
	const float zRot = sinf(__state.subjectRotation.y * 1.5f);
	model = glm::rotate(model, __state.subjectRotation.x * 1.5f, glm::vec3(xRot, 0.f, zRot));
	
	view = glm::lookAt(__state.camPosition, __state.subjectPosition + __state.subjectOffset,
				 __state.camUp);
	
	const float ASPECT_RATIO = static_cast<float>(__state.mainWindowHeight) /
					 static_cast<float>(__state.mainWindowWidth);

	projection = glm::perspective(glm::radians(45.f), (1.f / ASPECT_RATIO), 0.01f, 100.f);
}
```

The next step was to retrieve rgb and y values from the screen. A quick search gave the answer immediately: OpenGL has built-in functions for this and so it proved the easiest piece of the entire project to implement:

```cpp
float* capturedColor = new float[in_width * in_height * 3];
float* capturedDepth = new float[in_width * in_height];
glReadPixels(0, 0, in_width, in_height, GL_RGB, GL_FLOAT, capturedColor);
glReadPixels(0, 0, in_width, in_height, GL_DEPTH_COMPONENT, GL_FLOAT, capturedDepth);
```

Since SBT has no real save functions (that's how Sean Barrett wants it) I wrote my own, saving rgb as .tga and depth as .pgm. Pgm *is* a rather obscure format and it produces eye watering size on disk but it does support 16-bit grayscale and can be loaded by Photoshop and Gimp, which were the main objectives.

Now I had a working prototype, run from the command line, that did everything it was supposed to do. However, starting up the program with arguments for files every time you want to load a new one quickly gets tedious, especially when textures are in different folders from the meshes. So it was time to add a couple of features: scanning for mesh files, finding their textures, and displaying the results as text in the editor. Thankfully, std::filesystem turned the file scanning into a doddle. With gltfs, there are file paths in the json and so I could use those to retrieve the texture I wanted. For fbx and obj I just had to assume the texture would be in the same dir as the mesh and that its filename would contain the mesh filename.

Text is a different matter. I read a description of how it is usually done: "Draw a quad for each letter and put a texture featuring the correct letter on it." and I figured that was something I should be able to do. For me, this has been a defining moment in my progress to become a *real* programmer. No tutorials, only this description, and having worked with data arrays for weeks, I saw a way of accomplishing it. First, I created the texture, taking an embarrassing amount of time to try different fonts in different sizes (in my defence, I didn't know beforehand which ones were monospaced).  

![The text starts here](img/portfolio/M2I/bmtxt.png "All the letters a programmer could ever want!")

I then created a struct for the uv-coordinates and a map (that's a dictionary for you C# coders out there) with the corresponding chars:

```cpp
void GetCascadiaMap(std::map<char, BMuv> &bmuv)
{
	bmuv.clear();
	int asciiValue = 32;
	int tableWidth = 32;
	int tableHeight = 3;
	for (int i = 0; i < 3; i++)
	{
		for (int j = 0; j < 32; j++)
		{
			char c = char(asciiValue);
			BMuv tempbmuv;
			tempbmuv.topLeftUV = { static_cast<float>(j) / tableWidth ,
						static_cast<float>(i) / tableHeight };
			tempbmuv.topRightUV = { static_cast<float>(j + 1) / tableWidth ,
						 static_cast<float>(i) / tableHeight };
			tempbmuv.bottomLeftUV = { static_cast<float>(j) / tableWidth ,
						 static_cast<float>(i + 1) / tableHeight };
			tempbmuv.bottomRightUV = { static_cast<float>(j + 1) / tableWidth ,
						 static_cast<float>(i + 1) / tableHeight };

			bmuv.insert({ c, tempbmuv });
			
			asciiValue++;
		}
	}
}
```

which in turn was used to map the correct letters to the quads:

```cpp
const size_t TEXTLENGTH = std::strlen(text);
vertices.clear();
vertices.resize(TEXTLENGTH * 4);
const float quadWidth = 20.f / screenWidth;
const float quadHeight = 36.f / screenHeight;
const float zValue = 0.01f;

int pos_itr = 0;
int v_itr = 0;
for (int i = 0; i < TEXTLENGTH; i++)
{
	char c = text[i];

	if (static_cast<int>(c) < 32 || static_cast<int>(c) > 126)
		c = '#';

	const float lposx = pos_itr * quadWidth;
	const float rposx = (pos_itr + 1) * quadWidth;
	const float tposy = quadHeight;
	const float bposy = 0.f;
	const BMuv charUV = textmap.at(c);
	VertexText* v = vertices.data() + v_itr;

	v->position = { lposx, tposy, zValue };
	v->texCoords = charUV.topLeftUV;
	v->colour = colour;
	v++;

	v->position = { rposx, tposy, zValue };
	v->texCoords = charUV.topRightUV;
	v->colour = colour;
	v++;

	v->position = { rposx, bposy, zValue };
	v->texCoords = charUV.bottomRightUV;
	v->colour = colour;
	v++;

	v->position = { lposx, bposy, zValue };
	v->texCoords = charUV.bottomLeftUV;
	v->colour = colour;
	v++;
		
	pos_itr++;
	v_itr += 4;
}
```

Finally, a little gratiuitously, I created a macro

```cpp
// Requires textbuffer and textmap to be named as such. Expects a variable as second argument.
#define WRITE_VAR(text, v, row, colour)	{ glUniform1i(UNIFORM_LINE, row);\
					snprintf(textbuffer, BUFFER_SIZE, text, v);\
					textline.WriteLine(textbuffer, textmap, ETextColour:: colour); }
```

So that I could create text in the while-loop like this:

```cpp
WRITE_VAR("Far render: %f", __state.orthoFar, 0, RED)
```

And it all just worked.  

![Using m2i](img/portfolio/M2I/M2I_usage.gif "A little more to the left! Look sexy!")

There was just one more thing. I have mentioned a couple of times now how I wanted the program to be portable and without dependencies and yet now I had texture with characters to be able to write text in the editor. So, obviously I was going to inline it into an array! Before I had time to do it, Johno discovered and lamented the same thing, so I told him I was going to inline it to which there was hearty approval.

The good thing about text is that you only really need the alpha and you only need it as on or off, meaning I could reduce each pixel to a single bit. I wrote a program to load the texture and encode 8 pixels per byte before printing in the form of an array to a text file. Actually, I wrote a duplicate in binary form to a separate file for debugging purposes as well.

```cpp
std::vector<uint8_t> alpha;
for (int32_t i = 3; i < image_size; i += 4)
{
	alpha.push_back(image_data[i]);
}
std::ofstream testfile(".\\bitslayout.txt");
if (testfile.is_open())
{
	int32_t tick = 1;
	for (int32_t i = 0; i < alpha.size(); i++)
	{
		testfile << static_cast<int>(alpha[i] / 255);
		if (tick == 8)
		{
			testfile << ", ";
			tick = 0;
		}
		tick++;
	}
	testfile.close();
}
else
	std::cout << "Failed to create bitslayout.txt" << std::endl;

std::ofstream outfile(".\\array.txt");
if (outfile.is_open())
{
	int tick = 1;
	for (int32_t i = 0; i < alpha.size(); i += 8)
	{
		uint8_t byte = 0;
		uint8_t mask =	(alpha[i + 7] / 255	<< 0) |
				(alpha[i + 6] / 255	<< 1) |
				(alpha[i + 5] / 255	<< 2) |
				(alpha[i + 4] / 255	<< 3) |
				(alpha[i + 3] / 255	<< 4) |
				(alpha[i + 2] / 255	<< 5) |
				(alpha[i + 1] / 255	<< 6) |
				(alpha[i]     / 255	<< 7);
		byte = byte | mask;
		
		outfile << static_cast<int>(byte) << ", ";
	}
	outfile.close();
}
else
	std::cout << "Failed to create array.txt" << std::endl;
```

Then all I had to do was delete the last comma, copy the text file into an array and write a function that decodes it.

```cpp
uint8_t* CascadiaData()
{
	for (int32_t i = 0; i < cascadia_elements; i++)
	{
		for (int32_t a = 0; a < 8; a++)
		{
			for (int32_t rgb = 0; rgb < 3; rgb++)
				textdata[(i * 32) + (a * 4) + rgb] = 255;

			uint8_t value = cascadiadata[i] << a;
			textdata[(i * 32) + (a * 4) + 3] = 255 * (value >> 7);
		}
	}

	return &textdata[0];
}
```

This project really made me feel like low level programming is all about structuring your for-loops but if that is so, then that is ok.

This project is available on https://github.com/B0realian/MeshToImage