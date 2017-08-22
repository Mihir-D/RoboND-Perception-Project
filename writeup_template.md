## Project: Perception Pick & Place
### Writeup Template: You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---


[//]: # (Image References)

[image1]: ./Images/with_noise.png
[image2]: ./Images/noisy_image.png
[image3]: ./Images/noise_removed.png
[image4]: ./Images/objects.png
[image5]: ./Images/table.png
[image6]: ./Images/pcl_cluster.png
[image7]: ./Images/confusion_matrix.png
[image8]: ./Images/object_recognition_test_scene1.png
[image9]: ./Images/object_recognition_test_scene3.png
[image10]: ./Images/picked1.png
[image11]: ./Images/picked2.png
[image12]: ./Images/picked3.png
[image13]: ./Images/picked4.png
[image14]: ./Images/picked5.png



# Required Steps for a Passing Submission:
1. Extract features and train an SVM model on new objects (see `pick_list_*.yaml` in `/pr2_robot/config/` for the list of models you'll be trying to identify). 
2. Write a ROS node and subscribe to `/pr2/world/points` topic. This topic contains noisy point cloud data that you must work with.
3. Use filtering and RANSAC plane fitting to isolate the objects of interest from the rest of the scene.
4. Apply Euclidean clustering to create separate clusters for individual items.
5. Perform object recognition on these objects and assign them labels (markers in RViz).
6. Calculate the centroid (average in x, y and z) of the set of points belonging to that each object.
7. Create ROS messages containing the details of each object (name, pick_pose, etc.) and write these messages out to `.yaml` files, one for each of the 3 scenarios (`test1-3.world` in `/pr2_robot/worlds/`).  See the example `output.yaml` for details on what the output should look like.  
8. Submit a link to your GitHub repo for the project or the Python code for your perception pipeline and your output `.yaml` files (3 `.yaml` files, one for each test world).  You must have correctly identified 100% of objects from `pick_list_1.yaml` for `test1.world`, 80% of items from `pick_list_2.yaml` for `test2.world` and 75% of items from `pick_list_3.yaml` in `test3.world`.
9. Congratulations!  Your Done!

# Extra Challenges: Complete the Pick & Place
7. To create a collision map, publish a point cloud to the `/pr2/3d_map/points` topic and make sure you change the `point_cloud_topic` to `/pr2/3d_map/points` in `sensors.yaml` in the `/pr2_robot/config/` directory. This topic is read by Moveit!, which uses this point cloud input to generate a collision map, allowing the robot to plan its trajectory.  Keep in mind that later when you go to pick up an object, you must first remove it from this point cloud so it is removed from the collision map!
8. Rotate the robot to generate collision map of table sides. This can be accomplished by publishing joint angle value(in radians) to `/pr2/world_joint_controller/command`
9. Rotate the robot back to its original state.
10. Create a ROS Client for the “pick_place_routine” rosservice.  In the required steps above, you already created the messages you need to use this service. Checkout the [PickPlace.srv](https://github.com/udacity/RoboND-Perception-Project/tree/master/pr2_robot/srv) file to find out what arguments you must pass to this service.
11. If everything was done correctly, when you pass the appropriate messages to the `pick_place_routine` service, the selected arm will perform pick and place operation and display trajectory in the RViz window
12. Place all the objects from your pick list in their respective dropoff box and you have completed the challenge!
13. Looking for a bigger challenge?  Load up the `challenge.world` scenario and see if you can get your perception pipeline working there!

## [Rubric](https://review.udacity.com/#!/rubrics/1067/view) Points
### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
### Writeup / README

#### 1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  

You're reading it!

`Note: Please refer to code [**object_recognition.py**](./pr2_robot/scripts/object_recognition.py) and NOT project_template.py.`

### Exercise 1, 2 and 3 pipeline implemented. These comprise the pcl_callback() function in object_recognition.py.
#### 1. Complete Exercise 1 steps. Pipeline for filtering and RANSAC plane fitting implemented.

I have implemented following steps from Exercise 1:

**1. Convert ROS message to PCL**

**2. Implement outlier filter to remove noise in camera image:** 
As the image obtained from camera has a lot of noise (see the camera image published on topic name???), I implemented the outlier removal filter to remove noise. To determine number of neighboring points to be analyzed, I tried different values in range 10-100 and found 30 to be suitably working. I had to reduce the threshold scale factor (i.e. standard deviation) to 0.01 as the noise was quite dominant. Refer the images below:

**Original Camera Image:**

![alt text][image2]

**After filtering:**

![alt text][image3]


**3. Perform Voxel Downgrid sampling to eliminate redundant computations:** 
I initially started with a Leaf size of 0.01. It would work well for the `test1.world`, but misidentified one or more objects in the other two test worlds and also couldn't identify one object in `test3.world`. I then reduced the leaf size till 0.003, at which all objects in all test scenes were recognized correctly. Refer image shown in step no. 7.

**4. Implement passthrough filter in Z direction to focus only on the table and object scene:** 
I implemented this filter such that the output of the filter contained only the table and objects on the table. (It eliminates everything else e.g. floor etc. which lies outside of the band of passthrough filter).

**5. Impement passthrough filter in X direction to remove edge of the table:** 
This filter removes the edge of the table which otherwise would appear along with objects when RANSAC plane segementation is performed for table plane. (note: similar results can also be obtained by finding an optimum lower limit for filtering performed in step 4.)

**6. RANSAC plane segmentation is performed:**
I performed RANSAC plane segmentation to separate table from the objects. 

**7. The inliers (table) and the outliers (objects) are extracted from RANSAC plane segmentation:** 
The inliers of the RANSAC filtering output are the table points whereas the outliers are all objects points, as the object planes are not parallel to table plane. Thus table cloud and objects point cloud are extracted.

Effect of filtering through steps 3-7:

**Objects Only**

![alt text][image4]

**Table Only**

![alt text][image5]

#### 2. Complete Exercise 2 steps: Pipeline including clustering for segmentation implemented.  

**8. Euclidean clustering is performed to detect separate object clusters:** 
I found the appropriate range for cluster size for Euclidean clustering. For cluster size less than 5000, some large objects were getting ommitted in the filtering. Hence the upper limit is 5000. Also, lower limit if increased from 100 would eliminated the small objects after filtering.

**9. Using cluster-masking, each cloud containing unique color is created. This helps to visualize the distincly identified objects in Euclidean clustering. See Image below**

![alt text][image6]


**10. Point cloud with noise removed, Objects' point cloud, Table point cloud and Detected objects' cloud are converted to ROS message type and published on respective ROS topics.**

#### 2. Complete Exercise 3 Steps.  Features extracted and SVM trained.  Object recognition implemented.

#### Feature Extraction and training SVM:

**1. RGB vs HSV:** 
I changed the initial RGB feature extraction to HSV which improved the accuracy from about 60% to about 75%, without changing any other parameters.

**2. Number of times the feature is captured for each object:** 
Initially, this value was 5. Changing it upto 10 only decreased the accuracy of SVM. I started seeing a good improvement with values greater than 20. For 30, I got accuracy of about 85%. With 50, it touched 90%. Finally, I used 100, which gave me an accuracy of 99% for test world 1 objects, 96% with test world 2 objects and 95% with test world 3 objects. (All these values are with 'linear' kernel)

**3. 'linear' vs 'rbf':** 
With rbf kernel, I saw a slight decrease (2%-3%) in accuracy in all cases compared to linear kernel. Hence, I have used a 'linear' kernel for training SVM.


**Training Accuracy  and Confusion Matrix:**

![alt text][image7]


#### Object Recognition:

**1. Extracted the color and normal Histogram features for each individual object extracted in Exercise 2.**

**2. From the classifier obtained from model.sav file, the object is predicted. All these recognized objects are appended in a list detected_objects, and published on topic `/Detected_objects`**


**Object Recognition Test Scene 1:**

![alt text][image8]


**Object Recognition Test Scene 3:**

![alt text][image9]



### Pick and Place Setup

#### 1. For all three tabletop setups (`test*.world`), perform object recognition, then read in respective pick list (`pick_list_*.yaml`). Next construct the messages that would comprise a valid `PickPlace` request output them to `.yaml` format.


Refer to code [`object_recognition.py`](./pr2_robot/scripts/object_recognition.py). The code is able to recognize 100% of the objects in all test scenarios. The pcl_callback() function in the code is already explained above. The recognized object list in pcl_callback() function is then passed to function `pr2_mover()`. 

Below I'll explain steps performed in function `pr2_mover()`.

1. Get parameters from the parameter server '/object_list' which gives the sequence in which the objects are to be picked.

2. For all the the parameters obtained, perform steps from step 3 to step 10.

3. Check if the name of the object in paramter list is detected.

4. If the name is not found, go to step 3 with next parameter in the parameter list.

5. Else if name is found in the detected object list, continue with steps below.

6. Calculate the centroid of the object.

7. Obtain other required parameters of the object such as test scene number, Object name, arm name (left or right), pick position and  place position.

8. Store all the parameters above in corresponding ROS datatypes.

9. Make a yaml dictionary list to store the above paramters with the help of make_yaml_dict() function.

10. Call the service pick place routine and pass all the object paramters.

11. Save the yaml dictionary list on computer. The output YAML files are stored [`here`](./outputs).


#### See below images in which the robot performs object pick and place tasks:

![alt text][image10]

![alt text][image11]

![alt text][image12]

![alt text][image13]

![alt text][image14]
