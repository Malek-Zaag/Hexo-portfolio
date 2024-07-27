---
title: Securinets Friendly CTF 2K22 forensics writeup
date: 2022-09-12 18:50:05
tags:
categories: [CTF Writeups]
---

# Securinets Friendly CTF 2K22 forensics writeup

This is my first CTF writeup for two forensics tasks in the Securinets Friendly CTF which was held online from the 30th of September 2022 to the 4th of October, how you like it .

## 4 in a row

![](https://cdn-images-1.medium.com/max/2000/1*7vuY2sZl8Hjqb4fTVpxiGA.png)

The challenge was given as a 150x150 png file and as the description mentioned it was about png color channels and specifically the alpha value of a pixel.

![chall.png](https://cdn-images-1.medium.com/max/2000/1*ssQVJYvTf-gig6y0knbCEQ.png)_chall.png_

We can read from the description that the flag was hidden in the alpha value of every pixel that its order is divisible by 4 . But before hacking into the challenge, letÔÇÖs talk about the png color channel and how the data is embedded in each pixel.Each pixel in an image is represented by the model RGBA(red,green,blue,alpha) and these pixels are presenting the image as a matrix.

So the description says that the author is hiding the flag in the diagonal pixels and he gives the formula to extract it , so i concluded with a script doing it.

```python
> from PIL import Image
> image = Image.open("00000000.png").convert("RGBA")
> pixeldata = list(image.getdata())
> flag=[]
> for i,pixel in enumerate(pixeldata):
> if i % 4 ==0:
> flag.append(255 * pixel[3])
> res=[chr(flag[i]) for i in range(len(flag)) if flag[i]!= 0]
> print(res)
> joined= ""    .join(res)
> print(joined)
> image.putdata(pixeldata)
> image.save("output.png")
```

and the flag was Securinets{PiLl0W_Pyth0N_1S_Awe5oMe}

## 5 in a row

![](https://cdn-images-1.medium.com/max/2000/1*xyEp9a9bkLcQFylZGVhUNg.png)

The second challengee was given as a 150x150 png file and as the description mentioned it was about png color channels and specifically the alpha value of a pixel.

![chall.png](https://cdn-images-1.medium.com/max/2000/1*ssQVJYvTf-gig6y0knbCEQ.png)_chall.png_

We can read from the description that the flag was hidden in the alpha value of every pixel that its order is divisible by 5 and i == j (diagonal of the image) . But before we jump to the extracting part letÔÇÖs talk about the png color channel and how the data is embedded in each pixel.Each pixel in an image is represented by the model RGBA(red,green,blue,alpha) and these pixels are presenting the image as a matrix.

So the description says that the author is hiding the flag in the diagonal pixels and he gives the formula to extract it , so i concluded with a script doing it.

```python
> from PIL import Image
> image = Image.open("00000000.png").convert("RGBA")
> pixeldata = list(image.getdata())
> px = image.load()
> flag=""
> for i in range(150):
> for j in range(150):
> if (i==j) and (i % 5 ==0):
> flag+=chr(255 \* px[i,i][-1]^(px[i,i][0]^((px[i,i][1] + px[i,i][2])%3)))
> print(flag)
> image.save("output.png")
```

and the flag was Securinets{AlpH4_1S_vEry_H4rD}
