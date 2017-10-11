## Project: Perception Pick & Place


[//]: # (Image References)

[image1]: ./data/normalized_model_1.png
[image2]: ./data/normalized_model_2.png
[image3]: ./data/normalized_model_3.png
[image4]: ./data/scene_1.png
[image5]: ./data/scene_2.png
[image6]: ./data/scene_3.png

### Exercise 1

First we remove pepper noise from the cloud 
```python
outlier_filter = pcl_data.make_statistical_outlier_filter()
k = 20
outlier_filter.set_mean_k(20)
x = 0.3
outlier_filter.set_std_dev_mul_thresh(x)
cloud_denoised = outlier_filter.filter()
```
where k is the number of neighbouring points used in the filtering process and x is the standard deviation threshold.

Then downsampling the cloud using a leafsize 0.005
```python
vox = cloud_denoised.make_voxel_grid_filter()
LEAF_SIZE = 0.005
vox.set_leaf_size(LEAF_SIZE, LEAF_SIZE, LEAF_SIZE)
cloud_filtered = vox.filter()
```
Removing redundant areas from the cloud  
```python
passthrough = cloud_filtered.make_passthrough_filter()
filter_axis = AXIS
passthrough.set_filter_field_name(filter_axis)
axis_min = 0.6
axis_max = 1.1
passthrough.set_filter_limits(axis_min, axis_max)
cloud_filtered = passthrough.filter()
```
The function was used using z and y for the variable AXIS so that we were only left with the table and objects.


Finally the filtered cloud was segmented using RANSAC plane fitting algorithm into two different set of points named extracted_inliers and extracted_outliers which contained the table and the objects respectively.
In the function max_distance was set so that the front of the table (vertical plane w.r.t camera) was also correctly segmented into inliers.
```python
seg = cloud_filtered.make_segmenter()
seg.set_model_type(pcl.SACMODEL_PLANE)
seg.set_method_type(pcl.SAC_RANSAC)
max_distance = 0.04
seg.set_distance_threshold(max_distance)

inliers, coefficients = seg.segment()
extracted_inliers = cloud_filtered.extract(inliers, negative=False)
extracted_outliers = cloud_filtered.extract(inliers, negative=True)
```
### Exercise 2

Euclidean clustering (also called DBSCAN) was used to cluster 3D points into different groups based on the spatial arrangement on the table
```python
white_cloud = XYZRGB_to_XYZ(extracted_outliers)
tree = white_cloud.make_kdtree()
ec = white_cloud.make_EuclideanClusterExtraction()
ec.set_ClusterTolerance(0.05)
ec.set_MinClusterSize(30)
ec.set_MaxClusterSize(2000)
# Search the k-d tree for clusters
ec.set_SearchMethod(tree)
# Extract indices for each of the 
```
Minimum and maximum number of points in a cluster was set to 30 and 2000. Maximum distance between neighbouring points in a cluster was set to 0.05. 

### Exercise 3
Training of the SVM was done running the following commands:

```bash
$ roslaunch sensor_stick training.launch # build-up the environment
$ rosrun sensor_stick capture_features.py # generate test data
$ rosrun sensor_stick train_svm.py # train svm
```
In the training process each object was rotated to random pose 100 times. The SVM was trained using point normals and HSV color features.
Below are images of the normalized confusion matrices obtained after training:
![alt text][image1] ![alt text][image2] ![alt text][image3] 

## Pick and Place Setup
### Results
The Pick and Place system was run in three different configurations by runnig the following commands:
```bash
$ roslaunch pr2_robot pick_place_project.launch # summons the environment
$ rosrun pr2_robot project_node.py # starts the recognition pipeline
```
The file project_node.py contains the perception functionality which was mainly copied from Exercises 1, 2 and 3. In additions to perception pipeline
the file contains code for generating request messages for pick_and_place_server based on the output of the perception. The request was formed as a .yaml file.     
The output yaml files can be found from the root directory of RoboND-Perception-Project. 

Below are visualized the result of recognition pipleline. 

![alt text][image4] ![alt text][image5] ![alt text][image6]

In world 1 all the object were recognized successfully (3/3). 
In world 2 all the objects were classified correctly except the glue which was given the label soap. This is odd because 
if we look at the confusion matrix of the world 2 we can see that in the training process there was zero confusion between the soap
and the glue. Again in the world 3 all the objects one is classified correctly. Again the glue is incorretly classified but this into biscuits.

Overall the results are OK and meet the requirements of the project. To improve the results we could increase the number of training 
examples and also evaluate the system using different kernels for the SVM. In the current implementation I am using the default kernel (linear) and it would be interesting 
to test for instance the RBF kernel. In the future I am also planning to finish the optional parts of the project.


