---
layout: post
title: "HackTheBox Sigma Technology Write-Up"
categories: Write-Up
date: 2022-11-12
image: /images/sigma-technology/Hack-The-Box-logo.png
description: Sigma Technology is a medium difficulty misc challenge where we "fool" an image classifier into labeling an image of a dog as an airplane by only altering five individual pixels.
tags: [HTB, Write-Up, Python, Misc]
katex: true
markup: "markdown"
---

![](/images/console/Hack-The-Box-logo.png#center)

******

## Getting Started

![](/images/sigma_technology/pics/1.png#center)

We launch the challenge instance and are instructed to download **Sigma\_Technology.zip**. We are also given the IP address/port number **167.99.89.94:31617**. The zipped file contains **sigmanet.h5**, a [Hierarchical Data Format](https://en.wikipedia.org/wiki/Hierarchical_Data_Format), as well as **model.py**, a python file which contains the following:
```python
import numpy as np
from tensorflow.keras.models import load_model

class_names = [
    'airplane', 'automobile', 'bird', 'cat', 'deer', 'dog', 'frog', 'horse',
    'ship', 'truck'
]

class SigmaNet:
    def __init__(self):
        self.name = 'sigmanet'
        self.model_filename = 'sigmanet.h5'
        try:
            self._model = load_model(self.model_filename)
            print('Successfully loaded', self.name)
        except (ImportError, ValueError, OSError):
            print('Failed to load', self.name)

    def color_process(self, imgs):
        if imgs.ndim < 4:
            imgs = np.array([imgs])
        imgs = imgs.astype('float32')
        mean = [125.307, 122.95, 113.865]
        std = [62.9932, 62.0887, 66.7048]
        for img in imgs:
            for i in range(3):
                img[:, :, i] = (img[:, :, i] - mean[i]) / std[i]
        return imgs

    def predict(self, img):
        processed = self.color_process(img)
        return self._model.predict(processed)

    def predict_one(self, img):
        confidence = self.predict(img)[0]
        predicted_class = np.argmax(confidence)
        return class_names[predicted_class]
```
Visiting the address/port given to us in the browser displays this webpage:
![](/images/sigma_technology/pics/2.png#center)

Based on the challenge description, files and webpage, we know that the webapp is using an image classifier to label the image of the dog, and we can assume that **sigmanet.h5** contains the [Keras](https://www.tensorflow.org/guide/keras/save_and_serialize) model's architecture for that classifier. We will also assume that **model.py** is the code that is used to process and label images with this model.

We know that we need to "fool" the classifier into thinking this picture of a dog is another object by altering some of the image's pixel values. The webapp allows us to alter the red/green/blue values, in an (x,y) coordinate system, for five different pixels. Let's send some test changes through the webapp to see what happens and how it works.
![](/images/sigma_technology/pics/3.png#center)

The classifier still labeled the image as a dog, but we were able to find out how the webapp works and how our data is being sent. It is sending a **POST** request to the endpoint **/point-laser** with the new pixel values as **form data** in **JSON** format.
```json
{
  "p1": "1,+1,+1,+1,+1",
  "p2": "2,+2,+2,+2,+2",
  "p3": "3,+3,+3,+3,+3",
  "p4": "4,+4,+4,+4,+4",
  "p5": "5,+5,+5,+5,+5"
}
```
We can use this information to write a python script to automate an attack and communicate with the webapp.

******

## Research

After digging around on the web for a few minutes I found the paper [One Pixel Attack for Fooling Deep Neural Networks](/images/sigma_technology/1710.08864v7.pdf).
![](/images/sigma_technology/pics/OPA.png#center)

The paper goes into a lot of detail, most of which is over my head, but what I took from it is that knowing the predicted probability labels for the image might be enough feedback for us to tailor an attack to fool the classifier. Because we have the model file and the code used to classify images, we can download the dog image, alter pixels and locally run the classifier on it while accessing the probability labels. We can randomly alter one pixel at a time, in a loop,  trying to bump the probability for a selected object to a desired amount, then move on to another pixel. Hopefully we can get the probability for our selected object higher than all the other objects in five pixel changes or less, fooling the classifier. We can then send our altered pixel values to the webapp and hopefully get the flag. **This now becomes an optimization problem**.

******

## Code it up

First let's set up a [python3](https://www.python.org) environment with the dependencies we will need. Let's install [requests](https://requests.readthedocs.io/en/latest/) for the webapp communication, [tensorflow-gpu](https://www.tensorflow.org/) for accessing the classifier and model, and [pillow](https://python-pillow.org/) for image processing via [PyPi](https://pypi.org/)'s [pip](https://pip.pypa.io/en/stable/) command.
```shell
python -m pip install requests tensorflow-gpu pillow
```

We'll create the file **solve.py** and write a function to download the dog image. Next we'll write a function to choose a random pixel from the dog image (as a [numpy array](https://numpy.org/doc/stable/reference/generated/numpy.array.html)), giving it random red/green/blue values, return the new image numpy array, and for logging/display purposes the pixels that have been  changed. We will also extend the **SigmaNet** class from **model.py** to override the **predict\_one()** method so that it also returns the **confidence** (probability) values.
```python
"""solve.py"""

import shutil
import json
from typing import Tuple, List, Dict
from os import system
from sys import exit as sys_exit
from random import randint
import numpy
import numpy.typing as npt
import requests
from PIL import Image
from model import SigmaNet, class_names

URL = "http://167.99.89.94:31617"

# Numpy types
NDArrayInt = npt.NDArray[numpy.int_]
NDArrayFloat = npt.NDArray[numpy.float32]


class ExtendsSigmaNet(SigmaNet):
    """Extends the model.py SigmaNet class."""

    def predict_one(self, img: NDArrayInt) -> Tuple[str, NDArrayFloat]:
        """Override SigmaNet().predict_one() method,
        adding confidence values in the return."""
        confidence = self.predict(img)[0]
        predicted_class = numpy.argmax(confidence)

        return (class_names[predicted_class], confidence)


def download_image() -> None:
    """Download dog image from server."""
    resp = requests.get(URL + "/static/dog.png", stream=True, timeout=60)
    with open("dog.png", "wb") as outfile:
        shutil.copyfileobj(resp.raw, outfile)


def randomize(narr: NDArrayInt) -> Tuple[NDArrayInt, List[int]]:
    """Randomize random pixel RGB value and save to
    new image numpy array.

    Returns new image numpy array and new RGB values."""
    image: NDArrayInt = numpy.copy(narr)
    x_value = randint(0, 31)
    y_value = randint(0, 31)
    red = randint(0, 255)
    green = randint(0, 255)
    blue = randint(0, 255)
    image[x_value, y_value] = [red, green, blue]

    return (image, [x_value, y_value, red, green, blue])
```

Next we'll write our main attack loop function. The plan is to convert our image into a numpy array, then iteratively alter pixel values with **randomize()**, running the **ExtendsSigmaNet().predict\_one()** method on the resulting numpy array, attempting to increase confidence for the *airplane* label by a non-trivial amount. We start with a high threshold for an acceptable confidence level and after roughly 100 attempts reduce that threshold with a decreasing *multiplier* value until a desired confidence level is met. We then save that result and restart the process until the image is classified as an *airplane*. We will display information for each attempt along the way to monitor our progress and finally return the total number of attempts, classifier result, new image numpy array and a list of changed pixels.
```python
# pylint: disable=too-many-locals
def attack() -> Tuple[int, str, NDArrayInt, Dict[Tuple[int, int], List[int]]]:
    """Performs a One Pixel Attack on dog image, five times,
    each time trying to increase the prediction confidence
    for "airplane", until it's confidence is higher than any
    other object.

    Returns number of change attempts, final/new image prediction,
    final/new image numpy array, and dict of changed pixels."""

    # Convert image to numpy array
    img = Image.open("dog.png")
    np_img = numpy.array(img)[:, :, :3]

    # Initialization
    count = 0
    pixel_change = 0
    const = 50
    multiplier = const
    visited: List[Tuple[int, int]] = []
    changes: Dict[Tuple[int, int], List[int]] = {}
    sig_net: ExtendsSigmaNet = ExtendsSigmaNet()
    guess = sig_net.predict_one(np_img)
    old_plane_val: numpy.float32 = guess[1][0]
    target = old_plane_val * multiplier

    # Attack loop
    print("\n\nSCRIPT GO BRRRRRRRRRRRR...........\n")
    while guess[0] != "airplane" and pixel_change <= 4:
        system("clear")
        print("\nSCRIPT GO BRRRRRRRRRRRR...........\n")

        # For every 100 runs, decrease the confidence multiplier
        #  by one, therefore decreasing the overall target plane
        #  confidence. This way we can optimize by trying to get
        #  the largest confidence change first (since our changes
        #  are pseudo-random), then slowly lower our expectations
        if count % 100 == 99:
            multiplier -= 1

        # Generate new attempt
        new = randomize(np_img)
        x_value, y_value, red, green, blue = new[1]
        guess = sig_net.predict_one(new[0])
        count += 1

        # Print attempt info
        print("ATTEMPT:", count)
        print(f"TRYING: {x_value, y_value} {list((red, green, blue))}")
        print("PLANE CONFIDENCE MULTIPLIER:", multiplier)
        print("TARGET PLANE CONFIDENCE:", target)
        print("PIXEL CHANGES:", pixel_change)
        for key, value in changes.items():
            print(key, value)
        print("-----------------------")
        for index, value in enumerate(guess[1]):
            print(class_names[index], value)
        print()

        # New airplane confidence check
        if guess[1][0] >= target or guess[0] == "airplane":

            # Reset multiplier, save new image numpy array for next
            #  generation, record changed values and adjust new
            #  confidence target
            multiplier = const
            np_img = new[0]
            if (x_value, y_value) not in visited:
                visited.append((x_value, y_value))
                pixel_change += 1
            changes[(x_value, y_value)] = [red, green, blue]
            old_plane_val = guess[1][0]
        target = old_plane_val * multiplier

    return (count, guess[0], new[0], changes)
```

Now that the heavy lifting is done and we have presumably fooled the local classifier, we create a function to convert our new image numpy array back into a png image and save it. We build our POST request with our pixel value changes, send that data to the webapp and display some information, including the raw webpage text returned from our request, hopefully containing the challenge flag.
```python
def print_send_save(count: int, guess: str, new: NDArrayInt,
                    changes: Dict[Tuple[int, int], List[int]]) -> None:
    """When defeated, save new image, print
    values and send data to server."""

    # Save new image of dog that classifies as airplane
    if guess == "airplane":
        not_dog = Image.fromarray(new)
        not_dog.save("not_dog.png")

        # Print changed values and build request data string
        #  with changed values
        print("DEFEATING PIXEL VALUES:")
        data: Dict[str, str] = {}
        key_count = 1
        for key, value in changes.items():
            p_value = "".join([str(_) + ",+" for _ in key])
            p_value += "".join(str(_) + ",+" for _ in value)
            data["p" + str(key_count)] = p_value[:-2]
            key_count += 1
            print(key, value)
        if len(data) < 5:
            for i in range(len(data) + 1, 6):
                data["p" + str(i)] = data["p1"]

        # Write request data string of changed values to file
        with open("defeating_values.txt", "w", encoding='utf-8') as defeat:
            defeat.write(json.dumps(data))

        # Print attempt information and request data string, or failure
        print("\n")
        _r = requests.post(URL + "/point-laser", data=data, timeout=60)
        print(_r.text)
        print("\nATTEMPT:", count)
        print("GUESS:", guess)
        print("\nFORM DATA:", data)
    else:
        print("\nFAILED!!\n")
```

Finally, we call our functions when the script is run.
```python
if __name__ == "__main__":
    try:
        download_image()
        print_send_save(*attack())
    except KeyboardInterrupt:
        sys_exit()
```

******

## Running the Script

Here is a screenshot of our running script as it tries to get the image classified as an airplane.
![](/images/sigma_technology/pics/4.png#center)

Our script has successfully fooled the local classifier within just a few minutes, in under 5k attempts.
![](/images/sigma_technology/pics/6.png#center)

It then sends the five pixel value changes to the webapp and returns the raw webpage text, which includes the challenge flag.
![](/images/sigma_technology/pics/7.png#center)

******

## Just for Fun
We can also manually input the changes into the webapp through the browser and see the rendered webpage containing the flag.
![](/images/sigma_technology/pics/flag.png#center)

Then open the original dog image and the new "airplane" image to visually compare and see the differences.
![](/images/sigma_technology/pics/dog.png#center)

![](/images/sigma_technology/pics/not_dog.png#center)


******

## Conclusion

At the time of retirement this challenge only had 111 solves on HackTheBox. I'm not sure if people found it too difficult, were scared off by the machine learning aspect, or were just uninterested in challenges of this nature. I found it to be relatively medium difficulty for a misc challenge by just brute-forcing with some slight optimization, and had fun while developing a solution.


******


