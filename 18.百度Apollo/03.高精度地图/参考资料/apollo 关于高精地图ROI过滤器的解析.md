- [apollo 关于高精地图ROI过滤器的解析_xiaoma_bk的博客-CSDN博客_地图roi](https://blog.csdn.net/xiaoma_bk/article/details/122586900)

**总结一下高精地图ROI的过程**

1. 高精地图查询，获得路面路口的`polygons`多边形信息；
2. 点云坐标变换。将原始点云`cloud`从`lidar`坐标系转到`ENU`局部坐标系`cloud_local`；`polygons`从世界坐标系转到`ENU`局部坐标系`polygons_local`；
3. 将顶点存储形式的路面路口`polygon`信息，转换成填充(点阵)形式的存储方式。扫描线算法转换，`bitmap`存储；
4. 根据`ROI LUT`查询表，标记原始点云`cloud_local`是否在`ROI`内或外面。

**点云处理**

- 高精地图`ROI`过滤器是回调的第一个过程。该过程处理在`ROI`之外的激光雷达点，去除背景对象：如路边建筑物和树木等。剩余的点云留待后续处理。
- 从最开始的模块框架图可以看到，`LidarProcessSubnode`子节点接受的输入数据类型是`ROS`原始的点云数据类型，`sensor_msgs::PointCloud2`，简单地看一下这个数据结构，也可以参考官方文档 [PointCloud2](http://docs.ros.org/api/sensor_msgs/html/msg/PointCloud2.html)。
- `sensor_msgs::PointCloud2`与第一个版本`sensor_msgs::PointCloud`有一些区别，支持任意的N维数据，并且支持任意的基础数据类型，也支持密集存储。从上面的说明可以看到，`PointCloud2`可以支持`2D`数据结构，每个点`N`维，每行`X`个通道，共`H`列，可以存储图像信息等。`fields`存储了各个通道的名称，例如`x,y,z,r,g,b`等。在这里我们使用的`PointCloud2`类型每个通道只需要`xyz`三维即可，表示坐标系中的位置。
- 还有一个细节，激光雷达获取的点云是`ROS`原始的`sensor_msgs::PointClouds`类型，而实际处理过程中使用的更多的是`PCL`库的`pcl::PointCloud`类型，需要在代码中做一个转换，使用`pcl_conversions`的`pcl::fromROSMsg`和`pcl::toROSMsg`函数即可方便的实现相互转换。

## 数据转换与ROI生成

- 在进行高精地图ROI过滤的过程中，第一步是接收来自激光雷达的原始点云数据、设备id、时间戳ts等信息，并将这些信息存入上述SensorObject类中。存储过程的代码中值得关注的两个点分别是传感器到世界坐标系的转换矩阵velodyne_trans以及sensor_msgs::PointCloud2到PCL::PointCloud的转换。

  ```c
  /// file in apollo/modules/perception/obstacle/onboard/lidar_process_subnode.cc
  void LidarProcessSubnode::OnPointCloud(const sensor_msgs::PointCloud2& message) {
    ...
    /// get velodyne2world transfrom
    std::shared_ptr<Matrix4d> velodyne_trans = std::make_shared<Matrix4d>();
    if (!GetVelodyneTrans(kTimeStamp, velodyne_trans.get())) {
      ...
    }
    out_sensor_objects->sensor2world_pose = *velodyne_trans;
  
    PointCloudPtr point_cloud(new PointCloud);
    TransPointCloudToPCL(message, &point_cloud);
  }
  ```

- 当获得原始的点云数据并转换成PCL格式以后，下一步需要从点云中检索ROI区域，这些ROI区域包含路面与路口的驾驶区域。

  ```c
  // apollo/modules/perception/obstacle/onboard/lidar_process_subnode.cc
  void LidarProcessSubnode::OnPointCloud(const sensor_msgs::PointCloud2& message) {
    /// get velodyne2world transfrom
    if (!GetVelodyneTrans(kTimeStamp, velodyne_trans.get())) {
    	...
    }
    /// call hdmap to get ROI
    HdmapStructPtr hdmap = nullptr;
    if (hdmap_input_) {
      PointD velodyne_pose = {0.0, 0.0, 0.0, 0};  // (0,0,0)
      Affine3d temp_trans(*velodyne_trans);
      PointD velodyne_pose_world = pcl::transformPoint(velodyne_pose, temp_trans);
      hdmap.reset(new HdmapStruct);
      hdmap_input_->GetROI(velodyne_pose_world, FLAGS_map_radius, &hdmap);
      PERF_BLOCK_END("lidar_get_roi_from_hdmap");
    }
  }
  ```

- 路面与路口的驾驶区域需要查询高精地图来完成，该阶段首先使用`tf`进行坐标系的转换(`lidar`坐标系到世界坐标系的变换矩阵)，配合`velodyne_pose`计算得到`velodyne_pose_world`(`lidar`在世界坐标系中的坐标)，坐标系具体情况请参考Apollo坐标系研究。真正获取`ROI`使用的是`GetROI`函数。具体的路口，车道存储形式请参考高精地图模块。

- 简单分析一下GetVelodyneTrans函数，这个函数功能是产生lidar坐标系到世界坐标系的变换矩阵。实现过程我们可以先简要的看一下再做功能分析：

  ```c
  /// file in apollo/modules/perception/onboard/transform_input.cc
  bool GetVelodyneTrans(const double query_time, Eigen::Matrix4d* trans) {
    ...
    // Step1: lidar refer to novatel(GPS/IMU)
    geometry_msgs::TransformStamped transform_stamped;
    try {
      transform_stamped = tf2_buffer.lookupTransform(FLAGS_lidar_tf2_frame_id, FLAGS_lidar_tf2_child_frame_id, query_stamp);
    } 
    Eigen::Affine3d affine_lidar_3d;
    tf::transformMsgToEigen(transform_stamped.transform, affine_lidar_3d);
    Eigen::Matrix4d lidar2novatel_trans = affine_lidar_3d.matrix();
  
    // Step2 notavel(GPS/IMU) refer to world coordinate system
    try {
      transform_stamped = tf2_buffer.lookupTransform(FLAGS_localization_tf2_frame_id, FLAGS_localization_tf2_child_frame_id, query_stamp);
    } 
    Eigen::Affine3d affine_localization_3d;
    tf::transformMsgToEigen(transform_stamped.transform, affine_localization_3d);
    Eigen::Matrix4d novatel2world_trans = affine_localization_3d.matrix();
  
    *trans = novatel2world_trans * lidar2novatel_trans;
  }
  ```

- 点云数据由lidar获取，所以数据都是以激光雷达lidar参考系作为标准参考系，在查询高精地图的时候需要世界坐标系坐标。因此获取变换矩阵分为两步。 第一步获取激光雷达lidar坐标系到惯测单元IMU坐标系的变换矩阵；第二步，获取惯测单元IMU坐标系到世界坐标系变换矩阵。从上述的代码中我们明显可以看到有两部分相似度很高的代码组成:

  - 计算仿射变换矩阵`lidar2novatel_trans`，激光雷达`lidar`坐标系到惯测`IMU`坐标系(车辆坐标系)变换矩阵。这个矩阵虽然通过`ROS`的`tf`模块调用`lookupTransform`函数计算完成，但是实际是外参决定，在运行过程中保持不变。
  - 计算仿射变换矩阵`novatel2world_trans`，惯测单元`IMU`坐标系(车辆坐标系)相对于世界坐标系的仿射变换矩阵。
  - 计算仿射变换矩阵`lidar2world_trans`，最终两个矩阵相乘得到激光雷达`lidar`坐标系到世界坐标系的变换矩阵。

## 坐标变换

- Apollo官方文档引用：对于(高精地图ROI)过滤器来说，高精地图数据接口被定义为一系列多边形集合，每个集合由世界坐标系点组成有序点集。高精地图ROI点查询需要点云和多边形处在相同的坐标系，为此，Apollo将输入点云和HDMap多边形变换为来自激光雷达传感器位置的地方坐标系。

  ```c
  /// file in apollo/modules/perception/obstacle/onboard/lidar_process_subnode.cc
  void LidarProcessSubnode::OnPointCloud(const sensor_msgs::PointCloud2& message) {
    /// get velodyne2world transfrom
    ...
    /// call hdmap to get ROI
    ...
    /// call roi_filter
    PointCloudPtr roi_cloud(new PointCloud);
    if (roi_filter_ != nullptr) {
      PointIndicesPtr roi_indices(new PointIndices);
      ROIFilterOptions roi_filter_options;
      roi_filter_options.velodyne_trans = velodyne_trans;
      roi_filter_options.hdmap = hdmap;
      if (roi_filter_->Filter(point_cloud, roi_filter_options, roi_indices.get())) {
        pcl::copyPointCloud(*point_cloud, *roi_indices, *roi_cloud);
        roi_indices_ = roi_indices;
      } else {
        ...
      }
    }
  }
  ```

- 坐标转换和之后的`ROI LUT`构造与点查询的步骤都在`HdmapROIFilter`这个类里面完成。

- 这个阶段使用到的变换矩阵就是以上的lidar2world_trans矩阵。看了官方说明，并配合具体的代码，可能会存在一些疑惑。这里给出一些变换的研究心得。坐标变换的实现是在HdmapROIFilter::Filter函数中完成。具体的变换过程如下：

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/hdmap_roi_filter.cc
  bool HdmapROIFilter::Filter(const pcl_util::PointCloudPtr& cloud,
                              const ROIFilterOptions& roi_filter_options,
                              pcl_util::PointIndices* roi_indices) {
    Eigen::Affine3d temp_trans(*(roi_filter_options.velodyne_trans));
    std::vector<PolygonDType> polygons;
    MergeHdmapStructToPolygons(roi_filter_options.hdmap, &polygons);
    ...
    // Transform polygon and point to local coordinates
    pcl_util::PointCloudPtr cloud_local(new pcl_util::PointCloud);
    std::vector<PolygonType> polygons_local;
    TransformFrame(cloud, temp_trans, polygons, &polygons_local, cloud_local);
    ...
  }
  
  void HdmapROIFilter::TransformFrame(
      const pcl_util::PointCloudConstPtr& cloud, const Eigen::Affine3d& vel_pose,
      const std::vector<PolygonDType>& polygons_world,
      std::vector<PolygonType>* polygons_local,
      pcl_util::PointCloudPtr cloud_local) {
    ...
  }
  ```

- 注意点1: 上面代码中`MergeHdmapStructToPolygons`函数负责把路口和路面的点云并入到多边形集合`polygons`，这里`roi_filter_options`里面的数据都是经过高精地图查询得到的路口和路面信息，是基于世界坐标系的，所以结果合并后`polygons`也是世界坐标系的数据。

- 注意点2: 输入的cloud是基于lidar坐标系的点云数据，而下面代码还需要转换成cloud_local、polygons_local，按照注释解释是局部坐标系，那么这个局部坐标系到底是什么坐标系？如果看得懂TransformFrame函数，可以不难发现：这个所谓"local coordinate system"，其实跟lidar坐标系很相近，他表示以lidar为原点的ENU坐标系，这个坐标系是以X(东)-Y(北)-Z(天)为坐标轴的二维投影坐标系。在TransformFrame函数中，

  ```c
  igen::Vector3d vel_location = vel_pose.translation();
   Eigen::Matrix3d vel_rot = vel_pose.linear();
   Eigen::Vector3d x_axis = vel_rot.row(0);
   Eigen::Vector3d y_axis = vel_rot.row(1);
  ```

- vel_location是lidar坐标系相对世界坐标系的平移成分，vel_rot则是lidar坐标系相对世界坐标系的旋转矩阵。那么从lidar坐标系到世界坐标系的坐标变换其实很简单，假设在lidar坐标系中有一个坐标点P(x1,y1,z1)，那么该点在世界坐标系下的坐标P_hat为:P_hat = vel_rot * P + vel_location. 了解了这个变换，接下来观察cloud和polygons的变换代码：

  ```c
  polygons_local->resize(polygons_world.size());
  for (size_t i = 0; i < polygons_local->size(); ++i) {
    const auto& polygon_world = polygons_world[i];
    auto& polygon_local = polygons_local->at(i);
    polygon_local.resize(polygon_world.size());
    for (size_t j = 0; j < polygon_local.size(); ++j) {
      polygon_local[j].x = polygon_world[j].x - vel_location.x();
      polygon_local[j].y = polygon_world[j].y - vel_location.y();
    }
  }
  ```

- 开始的时候也是很奇怪，为什么最后变换的形式是`P_local = P_world - translation`. 后来经过研究猜测(有待后续深入阅读证实)路口和路面多边形信息只经过平移达到新的局部`ENU`坐标系，可以推测其实世界坐标系也是`ENU`坐标系，所以两个坐标系之间没有旋转成分，直接移除平移就可以从世界坐标系变换到局部`ENU`坐标系。

- 注意其实变换前后`polygons_world`和`polygons_local`的高度`z`是变化的，但是由于`polygons`被用来做`2D`投影网格`LUT`构建，所以对高度z这一维度并不关心，这些就不做`z`的变化；另外强度`i`始终不会变化。

- ```
  P_world = vel_rot * P_local + translation
  ```

  当vel_rot旋转成分为0时:

  ```
  P_local = P_world - translation
  ```

  ```c
  cloud_local->resize(cloud->size());
  for (size_t i = 0; i < cloud_local->size(); ++i) {
    const auto& pt = cloud->points[i];
    auto& local_pt = cloud_local->points[i];
    Eigen::Vector3d e_pt(pt.x, pt.y, pt.z);
    local_pt.x = x_axis.dot(e_pt);
    local_pt.y = y_axis.dot(e_pt);
  }
  ```

- 上述`cloud`变换代码再次验证了世界坐标系也是`ENU`坐标系类型的说法，局部`ENU`坐标系和`lidar`坐标系共原点但存在一个旋转角度，`lidar`坐标系到世界坐标系变换的旋转矩阵是`vel_rot`，那么到局部`ENU`坐标系的旋转矩阵也应该是`vel_rot`；其次共原点说明两个坐标系的平移矩阵其实是0。最终变换到局部`ENU`坐标系的公式就是：`P_hat = vel_rot * P + 0`。也就是上面看到的公式。

- 注意这里为什么只进行x和y坐标的转换，而没有进行高度z和强度i的转换？首先强度在任何坐标系下都是一样的，所以不用进行转换。其次cloud从lidar坐标系转换到以lidar为原点的ENU局部坐标系cloud_local，只有旋转没有平移成分，因为原点一样，xy轴构成的平面是同一个平面，所以高度是一样的，不需要变换z。

- 另外补充一点猜测世界坐标系也是ENU类型坐标系的证据：

  ```c
  /// file in apollo/modules/perception/traffic_light/onboard/hdmap_input.cc
  bool HDMapInput::GetSignals(const Eigen::Matrix4d &pointd, std::vector<apollo::hdmap::Signal> *signals) {
    auto hdmap = HDMapUtil::BaseMapPtr();
    std::vector<hdmap::SignalInfoConstPtr> forward_signals;
    apollo::common::PointENU point;
    point.set_x(pointd(0, 3));
    point.set_y(pointd(1, 3));
    point.set_z(pointd(2, 3));
    int result = hdmap->GetForwardNearestSignalsOnLane( point, FLAGS_query_signal_range, &forward_signals);
    ...
  }
  ```

- 在交通信号灯感知模块中，有一个功能是根据当前车的位置，去查询高精地图，获取该位置处的信号灯信息。上面的代码中`GetSignals`函数实现了这个功能，输入`pointd`就是我们上述使用到的`lidar2world_trans`变换矩阵，代码中`point`是`PointENU`类型的点，并设置为变换矩阵的平移成分去世界坐标系查询(实际是根据世界坐标系下面车坐标查询高精地图)。因此可以初步判断，世界坐标系是`ENU`类型坐标系。

- 最后有个问题，转换到局部`ENU`坐标系有什么作用，以及效果。简单地说一下`ENU`坐标系是带有方向性的，所以转换到该坐标系下的`polygeons_world`和`cloud_local`其实是有东南西北的意思，从Apollo官方文档可以看到效果如下，得到当前位置下`ROI`区域内外的一个矩形框，这时候路面和路口就有东西南北走向的意义。

- 思考

  ：为什么不在车辆坐标系(IMU坐标系)或者lidar坐标系下面进行接下去的分割操作？

  - (这两个坐标系的方向都参考车头的方向，是没有东南西北这些地理位置信息的)。

## ROI LUT构造与点查询

- [Apollo](https://so.csdn.net/so/search?q=Apollo&spm=1001.2101.3001.7020)官方文档引用：Apollo采用网格显示查找表（`LUT`），如下图将`ROI`量化为俯视图`2D`网格，以此决定输入点是在`ROI`之内还是之外。如图1所示，该`LUT`覆盖了一个矩形区域，该区域位于高精地图边界上方，以普通视图周围的预定义空间范围为边界。它代表了与`ROI`关联网格的每个单元格（如用1/0表示在`ROI`的内部/外部）。为了计算效率，Apollo使用扫描线算法和位图编码来构建`ROI LUT`。
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/5d912f5f6ba04299ad99def07c93e250.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAeGlhb21hX2Jr,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

- 如上图，蓝色线条标出了高精地图`ROI`的边界，包含路表与路口。红色加粗点表示对应于激光雷达传感器位置的地方坐标系原始位置。2D网格由8*8个绿色正方形组成，在`ROI`中的单元格，为蓝色填充的正方形，而之外的是黄色填充的正方形。

- 基于`ROI LUT`，查询每个输入点的关系使用两步认证。对于点查询过程，Apollo数据编译输出如下:

  - 检查点在`ROI LUT`矩形区域之内还是之外。
  - 查询`LUT`中相对于`ROI`关联点的相应单元格。
  - 收集属于`ROI`的所有点，并输出其相对于输入点云的索引。

- 以上是Apollo官方文档描述的原话，能了解大概的作用流程，但是对具体的细节难以掌握。本节我们将从代码解剖上来具体了解所谓的"`ROI LUT`构造与点查询"。简单地说这个环节的作用就是：将上述转换到局部`ENU`坐标系下的路面与路口`ROI`的二维信息映射到一张`2D`网格图中，网格图中0表示非路口路面区域，1表示路口与路面区域，最后判断[点云](https://so.csdn.net/so/search?q=点云&spm=1001.2101.3001.7020)cloud中哪些点在路面`ROI`内(方便做行人，车辆分割)

- 先解释一下一些基本信息概念：

  - 从上面映射到局部`ENU`坐标系的路口和路面点云信息，这些点云并不是覆盖所有路口路面区域的，而是路口与路面的凸包，也就是角点。`polygons_local`里面其实存储了路面与边界线的角点/轮廓信息。`cloud_local`是所有原始点云映射到`ENU`局部坐标系过后的信息。
  - 需要将原先路口与路面等多边形的角点存储模式(节省内存)转换到填充模式，最常用的算法是扫描线算法，具体请参考GitHub首页链接。
  - 这个填充模式需要用到一个填充的2D网格。网格的大小范围、网格之间的间距(扫描线之间的大小)等信息，由外部文件定义。关于2D网格如何节省存储开销，就用到了`bitmap`数据结构：

- 用户定义的参数可在配置文件`modules/perception/model/hdmap_roi_filter.config`中设置，`HDMap ROI Filter` 参数使用参考如下表格：

  | 参数名称    | 使用                                                         | 默认   |
  | ----------- | ------------------------------------------------------------ | ------ |
  | range       | 基于LiDAR传感器点的2D网格`ROI LUT`的图层范围 (也就是以`lidar`为中心、ENU坐标系下：东西方向x的统计范围，南北方向y的统计范围)，如(-70, 70)*(-70, 70) | 70.0米 |
  | cell_size   | 用于量化2D网格的单元格的大小，主方向上两条网格线之间的距离   | 0.25米 |
  | extend_dist | 从多边形边界扩展 `ROI`的距离                                 | 0.0米  |

- 明白了`ROI LUT`的作用后，我们将从代码一步步了解Apollo采用的方案。上面小节讲到使用`TransformFrame`函数完成原始点云到局部`ENU`坐标系点云的转换以后得到了`cloud_local`映射原点云，`polygons_local`映射路面与路口多边形信息。接下来做的工作就是根据`polygons_local`构建`ROI LUT`。构建的过程在`FilterWithPolygonMask`函数中开启。

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/hdmap_roi_filter.cc
  bool HdmapROIFilter::Filter(const pcl_util::PointCloudPtr& cloud,
                              const ROIFilterOptions& roi_filter_options,
                              pcl_util::PointIndices* roi_indices) {
    // 1. Transform polygon and point to local coordinates
    TransformFrame(cloud, temp_trans, polygons, &polygons_local, cloud_local);
  
    return FilterWithPolygonMask(cloud_local, polygons_local, roi_indices);
  }
  
  bool HdmapROIFilter::FilterWithPolygonMask(
      const pcl_util::PointCloudPtr& cloud,
      const std::vector<PolygonType>& map_polygons,
      pcl_util::PointIndices* roi_indices) {
    // 2. Get Major Direction as X direction and convert map_polygons to raw
    // polygons
    std::vector<PolygonScanConverter::Polygon> raw_polygons(map_polygons.size());
    MajorDirection major_dir = GetMajorDirection(map_polygons, &raw_polygons);
  
    // 3. Convert polygons into roi grids in bitmap
    Eigen::Vector2d min_p(-range_, -range_);
    Eigen::Vector2d max_p(range_, range_);
    Eigen::Vector2d grid_size(cell_size_, cell_size_);
    Bitmap2D bitmap(min_p, max_p, grid_size, major_dir);
    bitmap.BuildMap();
  
    DrawPolygonInBitmap(raw_polygons, extend_dist_, &bitmap);
  
    // 4. Check each point whether is in roi grids in bitmap
    return Bitmap2dFilter(cloud, bitmap, roi_indices);
  }
  ```

- 可以看到构建的过程总共分为3部分(其实2,3就能完成构建；4只是`check`，分类`cloud_local`中在路面`ROI`内和外的点云)。接下来逐个分析流程，第2步求`polygons_local`的主方向比较简单，只要计算多边形点云集合中，x/东西方向与y/南北方向最大值与最小值的差，差越大跨度越大。选择跨度小的方向作为主方向。(这部分代码比较简单，所以不再贴出来)。

- `bitmap`构建过程

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/bitmap2d.cc
  Bitmap2D::Bitmap2D(const Eigen::Vector2d& min_p, const Eigen::Vector2d& max_p, const Eigen::Vector2d& grid_size, DirectionMajor dir_major) {
    dir_major_ = dir_major;
    op_dir_major_ = opposite_direction(dir_major);
    min_p_ = min_p;
    max_p_ = max_p;
    grid_size_ = grid_size;
  }
  
  void Bitmap2D::BuildMap() {
    Eigen::Matrix<size_t, 2, 1> dims = ((max_p_ - min_p_).array() / grid_size_.array()).cast<size_t>();
    size_t rows = dims[dir_major_];
    size_t cols = (dims[op_dir_major_] >> 6) + 1;
    bitmap_ = std::vector<std::vector<uint64_t>>(rows, std::vector<uint64_t>(cols, 0));
  }
  ```

- 在构建`bitmap`的过程中，上面的代码对应`bitmap`初始化，主要做的工作就是设置参数，设置`min_p_，max_p_`，这两个参数就是对应外部文件的`range`参数，默认70米。`grid_size`是网格线之间的距离，默认0.25m。最后可以看到申请了一个`bitmap_的2D`向量，这个向量有`rows`行能理解，为什么列`cols`需要除以2^6次(64)呢？因为他是`bitmap`，一个`unsigned64`有64位，每一位bit存储一个网格0/1值，起到节省开销作用。所以`bitmap_`的大小如果是mxn，那么他可以存储mx64n网格大小的数据。

- 难点是`DrawPolygonInBitmap`函数，这也是主要工作完成的函数，在这个函数里面，会真正的构建`ROI LUT`。我们接着分析他的实现代码：

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/polygon_mask.cc
  void DrawPolygonInBitmap(const typename PolygonScanConverter::Polygon& polygon, const double extend_dist, Bitmap2D* bitmap) {
    ...
    // 1. Get valid x range
    Interval valid_x_range;
    GetValidXRange(polygon, *bitmap, major_dir, major_dir_grid_size, &valid_x_range);
  
    // 2. Convert polygon to scan intervals(Most important)
    std::vector<std::vector<Interval>> scans_intervals;
    PolygonScanConverter polygon_scan_converter;
    polygon_scan_converter.Init(major_dir, valid_x_range, polygon, major_dir_grid_size);
    polygon_scan_converter.ConvertScans(&scans_intervals);
  
    // 3. Draw grids in bitmap based on scan intervals
    double x = valid_x_range.first;
    for (size_t i = 0; i < scans_intervals.size(); x += major_dir_grid_size, ++i) {
      for (const auto& scan_interval : scans_intervals[i]) {
        ...
        bitmap->Set(x, valid_y_range.first, valid_y_range.second);
      }
    }
  }
  
  /// file in modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/polygon_scan_converter.cc
  void PolygonScanConverter::ConvertScans( std::vector<std::vector<Interval>> *scans_intervals) {
    scans_intervals->resize(scans_size_);
  
    DisturbPolygon();
    ConvertPolygonToSegments();
    BuildEdgeTable();
  
    // convert polygon to filled table
    ...
  }
  ```

- 从上面段代码我们来分析`DrawPolygonInBitmap`的流程，首先`GetValidXRange`函数获取主方向上，最大值和最小值，跟`MajorDirection`几乎无差异，这里不再分析。有基础的应该容易看懂。接着第二部分是将路面与边界线的多边形有序集合转换成`Edge`信息，如何转换？
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/171809cb7e6a473796ab59600870ae1f.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAeGlhb21hX2Jr,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

- 需要了解一个前提，`polygons_local`里面存储了路面和路口的多边形信息，对于每个多边形`polygon`，它里面存储了N个点云(主要使用其xy坐标)，物体的轮廓点是有序的排列的，所以每两个相邻的点可以构建一条边，最后得到一个封闭的轮廓(如上图A所示)。所以每个`polygon`里面存的形式为：
  `polygon.data: {P1.x, P1.y, P2.x, P2.y ,..., P7.x, P7.y, P8.x, P8.y}`

- 接下来，我们需要将这个角点存储形式彻底转换成填充(点阵)形式，转换的步骤是:

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/polygon_scan_converter.cc
  void PolygonScanConverter::DisturbPolygon() {
    for (auto &pt : polygon_) {
      double &x = pt[major_dir_];
      double d_x = (x - min_x_) / step_;   
      int int_d_x = std::round(d_x);          // get grid line id
      double delta_x = d_x - int_d_x;         // compute distance between point and grid line in major direction 
      if (std::abs(delta_x) < kEpsilon) {
        if (delta_x > 0) {                    // point in the right side of the grid line ==> right move fix(E.g. P2, P4)
          x = (int_d_x + kEpsilon) * step_ + min_x_; 
        } 
        else {                                // point in the left side of the grid line ==> left move fix(E.g. P1, P6)
          x = (int_d_x - kEpsilon) * step_ + min_x_; 
        }
      }
    }
  }
  ```

- 如上图C，`ConvertPolygonToSegments`函数将多边形相邻的两个点保存出他们的边`segment_`(以`pair<point,point>`形式存储)，同时计算得到这条边的斜率`slope_`，最后计算得到的`segment_`和`slope_`里面保存的信息方式为：

  - `segment_: {<P1,P2>, <P2,P3>, <P3,P4>, <P5,P4>, <P6,P5>, <P7,P6>, <P7,P8>}`;
  - `slope_: {0.2, 0.8, -4, 0.7, 0.4, -1, 10, 8}`

- 这里`segment_`里面每个元素两个点存储的顺序依赖其主方向上的坐标值，永远是后面一个点的坐标比前面一个点的坐标大。`segment_[n].first.x < segment_[n].second.x`，在代码中也相对比较简单。

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/polygon_scan_converter.cc
  void PolygonScanConverter::ConvertPolygonToSegments() {
    for (size_t i = 0; i < vertices_num; ++i) {
      const Point &cur_vertex = polygon_[i];                          // first point info
      const Point &next_vertex = polygon_[(i + 1) % vertices_num];    // second point info
      // store segment which segment_[i].first.x < segment_[i].second.x
      if (cur_vertex[major_dir_] < next_vertex[major_dir_]) {         
        segments_.emplace_back(cur_vertex, next_vertex);
      } else {
        segments_.emplace_back(next_vertex, cur_vertex);
      }
      // compute slope k
      double x_diff = next_vertex[major_dir_] - cur_vertex[major_dir_];    
      double y_diff = next_vertex[op_major_dir_] - cur_vertex[op_major_dir_];
      std::abs(cur_vertex[major_dir_] - next_vertex[major_dir_]) < kEpsilon
          ? slope_.push_back(kInf) : slope_.push_back(y_diff / x_diff);
    }
  }
  ```

- 如上图D，根据主方向上的`valid_range`(最大坐标和最小坐标差值)，以及网格线间距`grid_size`，画出`valid_range/grid_size`条网格线，然后根据`segment_`里面的两个点计算每条边跟其后面的网格线的第一个交点`E`。这里计算`E`有什么用？为什么只计算第一个交点`E`，而不计算边`S`和所有网格线可能的交点？

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/polygon_scan_converter.cc
  bool PolygonScanConverter::ConvertSegmentToEdge(const size_t seg_id, std::pair<int, Edge> *out_edge) {
    const Segment &segment = segments_[seg_id];
    double min_x = segment.first[major_dir_] - min_x_;        // bias from left boundary
    double min_y = segment.first[op_major_dir_];              // y coordinate
  
    int x_id = std::ceil(min_x / step_);                      // the next closest grid line
    out_edge->first = x_id;
    Edge &edge = out_edge->second;
    edge.x = x_id * step_;                                    // compute x of interaction
    edge.max_x = segment.second[major_dir_] - min_x_;         // max x of this segment. For any interaction P, if P.x > edge.max_x, P is out of this segment 
    edge.max_y = segment.second[op_major_dir_];
    edge.k = slope_[seg_id];
  
    if (std::isfinite(edge.k)) {                              // compute y of the interaction
      edge.y = min_y + (edge.x - min_x) * edge.k;
    } else {
      edge.y = min_y;
      if (edge.y > edge.max_y) {
        std::swap(edge.y, edge.max_y);
      }
    }
    if (std::isfinite(edge.k) && edge.max_x < edge.x) {       // return false if interaction is not cross the grid line which his k is finite
      return false;
    }
    return true;
  }
  ```

- 上面代码相对来说不是特别难理解，`min_x, min_y`是计算这个边`segment`到主方向最左端的距离以及次方向上的坐标。通过`x_id=ceil(min_x/step_)`可以求出边的起点和后面相交的网格线id，最后用`edge.x, edge.y`来存储这个交点坐标。这里的`edge.max_x，edge.max_y`有什么作用？他解决了上述"边与其他后续网格线的交点坐标怎么计算？"这个问题。通过`(edge.x, edge.y)`作为起点，加上斜率k，就能计算该条边和接下来网格线的交点。

- 比如已知边`segment`和下条网格线`x_id`的交点坐标为`(edge.x, edge.y)`，那么和下面第二条网格线`x_id+1`的交点坐标计算方法为：`(x, y) = (edge.x+step_, edge.y+k*step_)`。怎么知道这个交点在这条边上还是在边的延长线上(不属于这条边)呢，只要判断计算得到的`x`小于`edge.max_x`是否成立。若成立，边和`x_id+1`网格线有交点；否则无交点。

- 另外返回`false`的条件有何含义，“k有限大并且边与x_id号网络线交点超过了边的最大坐标(在边的延长线上，边外)”，这说明，这条边与`x_id`号网格线没有交点，也不是垂直边，无效边！
  ![在这里插入图片描述](https://img-blog.csdnimg.cn/5378a03672b04575b0b67d8ba76bba98.png?x-oss-process=image/watermark,type_d3F5LXplbmhlaQ,shadow_50,text_Q1NETiBAeGlhb21hX2Jr,size_20,color_FFFFFF,t_70,g_se,x_16#pic_center)

- 如上图E，第一个问题"计算E有什么用?" 首先我们要知道一个问题：如果多边形只包含凸或者凹，那么网格线穿过多边形，如何计算落在多边形`ROI`里面的区间？只要计算多边形和该网格线的交点，然后按照`y`的大小从小到大排列，必定是`2n`的交点`{P1,P2,P3,...,P2n}`，那么落入多边形的区间肯定是`[P1.y,P2.y] [P3.y, P4.y], .. , [P2n-1.y, P2n.y]`。这个可以从上图证实。那么对于3中的交点`E`可以排序，两两组合最终得到路面与路口区域落在该网格线上的区间。这部分代码有点难，可以慢慢体会：

  ```c
  /// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/polygon_scan_converter.cc
  void PolygonScanConverter::UpdateActiveEdgeTable(
      const size_t x_id, std::vector<Interval> \*scan_intervals) {
    size_t valid_edges_num = active_edge_table_.size();
    size_t invalid_edges_num = 0;
    // For each edege in active edge table, check whether it is still valid.
    // Stage 1, compute next interaction with later grid line, if out the line, erase in step3 by setting the slope infinite
    for (auto &edge : active_edge_table_) {
      if (!edge.MoveUp(step_)) {
        --valid_edges_num;
        ++invalid_edges_num;
        edge.y = kInf;
      }
    }
    // Stage 2, add the new edge into vector which compute directly from step 3.
    size_t new_edges_num = 0;
    for (const auto &edge : edge_table_[x_id]) {
      if (std::isfinite(edge.k)) {
        ++valid_edges_num;
        ++new_edges_num;
        active_edge_table_.push_back(edge);
      } else {
        scan_intervals->emplace_back(edge.y, edge.max_y);
      }
    }
    // Stage 3, remove the interactions which out of the segment(p.y greater than p.max_x)
    if (invalid_edges_num != 0 || new_edges_num != 0) {
      std::sort(active_edge_table_.begin(), active_edge_table_.end(),
                [](const Edge &a, const Edge &b) { return a.y < b.y; });
      active_edge_table_.erase(next(active_edge_table_.begin(), valid_edges_num), active_edge_table_.end());
    }
    // Stage 4, compute interval of grid line #x_id
    for (size_t i = 0; i + 1 < active_edge_table_.size(); i += 2) {
      double min_y = active_edge_table_[i].y;
      double max_y = active_edge_table_[i + 1].y;
  
      scan_intervals->emplace_back(min_y, max_y);
    }
  }
  ```

- 根据代码段暂时将`ROI`区间计算分为4个阶段，第一个阶段就是3中所讨论的问题，前面每条边开始已知跟后一条`x_id-1`网格线的交点，那么这些边跟后续网格线`x_id, x_id+1`的交点怎么计算(计算如上图的`E13，E53`)，这里通过`MoveUp(step_)`函数向后推演一个`step_`计算，函数跟上面讲得一致，最后返回时候超过边最大x,超过则丢弃，不超过保留。第二阶段增加是否有新的边由步骤3中直接计算得到(新加入如上图的`E1-E7`,)；第三阶段就是按照次方向y的值从小到大排列，并删除超出边的那些点；最后一个阶段就是计算落入多边形与`id_x`网格线相交的区间，

- 经过BCDE四步骤，就能将基于定点存储形式的路口与路面坐标转化成填充(点阵)形式。最后得到的结果是`vector<std::vector> *scans_intervals`形式的扫描结果，`2D`向量，每行对应一个扫描线；每行的`vector`存储对应网格线的路面`ROI`区间。最后就只要把`scans_intervals`这个扫描结果转换成`bitmap`的点阵就行了，也就是将`scans_intervals`放入`bitmap_`的即可。填充部分，代码相对比较简单，这里就省略了。可能以64位bit的形式进行存储有点点让人费解，不过没关系，相信有C++[数据结构](https://so.csdn.net/so/search?q=数据结构&spm=1001.2101.3001.7020)基础的你一定能理解。

- 最后要做的就是对原始点云`cloud_local`进行处理，标记点云中哪些点在`ROI`以外，哪些点在`ROI`以内，`ROI`区域内的点云可以供下一步行人，车辆等物体分割。

```c
/// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/polygon_scan_converter.cc
bool HdmapROIFilter::Bitmap2dFilter(const pcl::PointCloud<pcl_util::Point>::ConstPtr in_cloud_ptr,
    const Bitmap2D& bitmap, pcl_util::PointIndices* roi_indices_ptr) {
  roi_indices_ptr->indices.reserve(in_cloud_ptr->size());
  for (size_t i = 0; i < in_cloud_ptr->size(); ++i) {
    const auto& pt = in_cloud_ptr->points[i];
    Eigen::Vector2d p(pt.x, pt.y);
    if (bitmap.IsExist(p) && bitmap.Check(p)) {
      roi_indices_ptr->indices.push_back(i);
    }
  }
  return true;
}

/// file in apollo/modules/perception/obstacle/lidar/roi_filter/hdmap_roi_filter/bitmap2d.cc
bool Bitmap2D::IsExist(const Eigen::Vector2d& p) const {
  if (p.x() < min_p_.x() || p.x() >= max_p_.x()) {
    return false;
  }
  if (p.y() < min_p_.y() || p.y() >= max_p_.y()) {
    return false;
  }
  return true;
}

bool Bitmap2D::Check(const Eigen::Vector2d& p) const {
  Eigen::Matrix<size_t, 2, 1> grid_pt = ((p - min_p_).array() / grid_size_.array()).cast<size_t>();
  Eigen::Matrix<size_t, 2, 1> major_grid_pt(grid_pt[dir_major_], grid_pt[op_dir_major_]);

  size_t x_id = major_grid_pt.x();
  size_t block_id = major_grid_pt.y() >> 6;  // major_grid_pt.y() / 64, which grid line
  size_t bit_id = major_grid_pt.y() & 63;    // major_grid_pt.y() % 64

  const uint64_t& block = bitmap_[x_id][block_id];

  const uint64_t first_one = static_cast<uint64_t>(1) << 63;
  return block & (first_one >> bit_id);
}
```

- 从上面代码中不难看到其实针对每个基于`ENU`局部坐标系的点云`cloud`，根据其x和y去`bitmap`里面做`check`，第一`check`该点是否落在这个网格里面(`x:[-range,range], y:[-range,range]`)，这个检查由`isExist`函数完成；第二个`check`，如果该点在`LUT`网格内，那么`check`这个点是否在路面`ROI`内，只要检查其对应的网格坐标去`bitmap`查询即可，1表示在路面；0表示在路面外，这个检查由`Check`函数完成。最后cloud点云中每个点是否在路面`ROI`内全部记录在`roi_indices_ptr`内，里面本质就是一个向量，大小跟`cloud`里面的点云数量相等，结果一一对应。

## 点云筛选

- 高地图`ROI`过滤器最后一步就是过滤工作，去除背景点云(`ROI`区域以外点云)，便于下一步物体分割。点云筛选由`PCL`库`pcl::copyPointCloud`函数实现，`roi_indices`存储ROI区域内点云的id。