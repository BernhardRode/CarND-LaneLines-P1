#**Finding Lane Lines on the Road** 

##Writeup

[//]: # (Image References)

[image1]: ./examples/grayscale.jpg "Grayscale"

---

### Reflection

###1. Describe your pipeline. As part of the description, explain how you modified the draw_lines() function.

My pipeline consists of 6 steps and is defined in the function lane_finding_pipeline.

The function takes three arguments:

```
image
steering_angle = 0
horizon = 50
```

steering angle will modify the trapezoid, i'm using to create a box, where my interesting lane lines are. 

[image1]: ./steering.png "Steering"

last argument is horizon, which defines in percent, how much of the image is being part of the interesting image.

[image1]: ./horizon.png "Horizon"


* first I simply grayscale the image

```
grayscaled_image = grayscale(img)
```

* now i apply some gaussian blurring to get rid of some noise

```
kernel_size = 5 # kernel size must be odd
gaussian_blur_image = gaussian_blur(grayscaled_image, kernel_size)
```

* out of the blurred image, i extract the lines by using the canny transformation.

```
low_threshold = 100
high_threshold = 250
canny_image = canny(gaussian_blur_image, low_threshold, high_threshold)
```

* now create a trapezoid, where the interesting lane lines are, depending on horizon and steering angle.

```
interesting_region = get_trapezoid(canny_image, steering_angle, horizon)
interesting_region_image = region_of_interest(canny_image, interesting_region)
```

* now use hough_lines to find the lines inside of the interesting region.

```
rho = 2
theta = np.pi/180 
threshold = 35 
min_line_len = 40 
max_line_gap = 100
hough_lines_image = hough_lines(interesting_region_image, rho, theta, threshold, min_line_len, max_line_gap, horizon)
```

* inside of hough_lines, draw_lines is called, it is working in the following way:

  * define some variables

```
lines_left = []
m_left = []
lines_right = []
m_right = []
ysize = img.shape[0]
xsize = img.shape[1]           
thickness_left = 8
thickness_right = 8

x_left = xsize
y_left = 0
x_right = 0
y_right = 0
```
  * now iterate over all the received lines
```
for line in lines:
    for x1,y1,x2,y2 in line:    
```

  * skip all vertical lines

```    
if(x2 == x1):
    continue
```
  * calculate the slope
```
m = float((y2 - y1) / (x2 - x1))   
```
  * categorize the lines into left and right, depending on their slope
  * for right lines, get the point on the bottom right
  * for left lines, get the point on the bottom left
```        
if m > 0.5 and m < 5:
    lines_right.append(line)
    m_right.append(m)
    x_right = max(x_right, x1, x2)
    y_right = max(y_right, y1, y2)
elif m < -0.5 and m > -5:
    lines_left.append(line)
    m_left.append(m)
    x_left = min(x_left, x1, x2)
    y_left = max(y_left, y1, y2)
```  
  * calculate the average slope c for both sides
``` 
m_left_average = np.mean(m_left)
m_right_average = np.mean(m_right)
```  
  * calculate c for both sides
```     
c_left = y_left - ( x_left * m_left_average )
c_right = y_right - ( x_right * m_right_average )
```  
  * calculate the start and endpoints for the left and right lines 
``` 
y1_left = ysize
x1_left = int(( ( ysize - c_left ) / m_left_average ))
y2_left = int( ysize * horizon / 100)
x2_left = int(( ( y2_left - c_left ) / m_left_average ))

y1_right = ysize
x1_right = int(( ( ysize - c_right ) / m_right_average ))
y2_right = int( ysize * horizon / 100)
x2_right = int(( ( y2_right - c_right ) / m_right_average ))
```  
  * draw the lines
``` 
cv2.line(img, (x1_left, y1_left), (x2_left, y2_left), color, thickness=thickness_left)
cv2.line(img, (x1_right, y1_right), (x2_right, y2_right), color, thickness=thickness_left)
``` 
* Finally create the weighted image and return the output
```
α=0.8
β=1.0
λ=0.0
weighted_img_image = weighted_img(hough_lines_image, img, α, β, λ)

return weighted_img_image
```    

###2. Identify potential shortcomings with your current pipeline


As the lines go up to the defined horizon, i think in very sharp curves, the algorithm will get to its limits. 
Also in bad light situations or when we do not have enpugh contrast to identify lines. There will be possible shortcommings.

The result of weighted_img() could be returned directly, to save some footprint, but it is ok for this project.


###3. Suggest possible improvements to your pipeline

I think it would be possible to improve the yellow lane finding by converting the image into other colorspaces (HSV,...) because thre it would be easier to extrapolate the yellow color.

Another place for some improvements is on bright streets, where we do not have a lot of color difference between lines and street. I think it would be good, to play around with contrast and saturation to get a better result.