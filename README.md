# 基于or-tools解决物流调度问题(二)

> **发布时间**: 2024-02-02
> **标签**: #服务器, #数据库, #运维, #elasticsearch, #java, #spring boot, #大数据

---

## 📞 联系方式
> 💬 **微信：sen232sen** | 🚀 **专业定制各类管理系统** | ⭐ **源码获取/技术支持**

---

:::tips  
在上一篇中，我们解决了基本的车辆路线问题以及带有容载限制的车辆取送问题。下面我们来解决一个具体的业务需求，并完成具体的接口设计以及代码实现。  
:::

## 需求背景：

:::tips  
现在有一个调度任务，包含若干个发货单，每个发货单有一个**供应商** 和一个**客户** ，相当于发货单的起点和终点。不同的发货单可以有**相同** 的起点或终点。我们需要对这些发货单上的起点和终点进行排线，找出**总里程最短** 的配送方案。  
最终的配送方案可以包含多个配送路线，每一条配送路线可以包含多个单据，每一张单都是原子性的，不可再分割。  
每一条配送路线对应一辆车，如果一辆车需要配送多张单，需要保证每张发货单的**取送顺序** ，即需要先经过起点取货才能到终点送货，在配送完成之后，不需要车辆返回起点。还需要考虑的是车辆的容量限制，一辆车的容量是有限的。我们只需要根据发货单中商品的数量计算容量即可。  
最终，我们需要计算出一共需要安排几条路线(车辆)来完成该调度任务，并且需要提供每条路线中经过的起点和终点的顺序，以及每条路线安排了哪些发货单。  
:::  
根据以上需求可以知道，我们还需要考虑不同的发货单可以有相同的起点和终点，并且还需要将实际的业务场景中的真实的坐标映射为or-tools所需要的数据结构，在计算完了之后再将虚拟的下标路径转换为真实的坐标路径。

## 接口设计

下面我们来看一下具体如何设计接口。首先我们需要明确的是：调度任务计算显然是一个非常复杂的过程，需要消耗大量的资源，因此计算这个过程必须是异步的。因此我们至少需要定义两个接口：创建调度任务、查询返回结果

### 创建调度任务

#### 请求参数

    {
    	"list": [
    		{
    			"deliveryCode": "",  //发货单号
    			"demands": 0,       //商品数量
    			"fromNode": "",		//发货起点
    			"toNode": ""		//发货终点
    		}
    	]
    }

这里的起点和终点，我们需要在数据库中维护一张表来记录他们的经纬度左边以及名称等信息。我们在数据库模型中会详细提到

#### 返回结果

    {
    	"dispatchId": 0 //调度任务编码
    }

由于是异步任务，所以这里创建了调度任务之后我们将任务编码返回即可

### 查询调度结果

#### 请求参数

    {
    	"dispatchId": 0 //调度任务编码
    }

#### 返回结果

    {
    	"data": [
    		{
    			"deliveryCode": "",  //该路线负责运送的发货单号通过,分隔
    			"routeDistance": 0,  //该路线总里程
    			"routeId": 0,				 //路线编号
    			"routeList": [			 //路线中的节点列表
    				{
    					"name": "",		   //节点名称
    					"latitude": "",  //纬度
    					"longitude": "", //经度
    					"nodeId": "",		 //节点ID
    					"nodeSeq": 0		 //节点序号
    				}
    			],
    			"dispatchId": 0			//调度任务编码
    		}
    	]
    }

每一个调度计划包含多条路线，每一条路线可以承载多个发货单号，并且包含多个节点，我们用nodeSeq这个字段来记录每个点的配送顺序即可

## 数据库设计

在上述的接口设计中，node节点的经纬度是不需要前端传入的，显然需要我们自己维护

#### node节点表

    CREATE TABLE `node` (
      `nodeId` varchar(32) NOT NULL COMMENT '主键Id',
      `name` varchar(200) DEFAULT NULL COMMENT '名称',
      `longitude` varchar(64) NOT NULL COMMENT '经度',
      `latitude` varchar(64) NOT NULL COMMENT '纬度'
      PRIMARY KEY (`nodeId`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='节点表';

记录每个节点的名称、经纬度即可，我们后续取出来计算距离的时候使用经纬度就能算出两两之间的距离

#### 调度任务表

    CREATE TABLE `scheduling` (
      `scheId` int(11) NOT NULL AUTO_INCREMENT COMMENT '调度任务编码',
      `state` int(11) NOT NULL DEFAULT '1' COMMENT '调度状态：1-待计算，2-计算完成，3-计算失败，4-计算超时,5-计算终止',
      `createTime` datetime DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
      `finishTime` datetime DEFAULT NULL COMMENT '完成时间'
      PRIMARY KEY (`scheId`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='调度任务表';

在总表中，我们记录调度任务的状态、创建时间和完成时间即可。这样我们在创建调度任务的时候，就可以往这张表里写入数据，然后开启异步计算任务，计算完成后再写入这张表即可。

    CREATE TABLE `scheduling_detail` (
      `scheId` int(11) NOT NULL COMMENT '调度任务编码',
      `deliverCode` varchar(32) NOT NULL COMMENT '发货单号',
      `srcNodeId` varchar(32) NOT NULL COMMENT '源节点编码',
      `dstNodeId` varchar(32) NOT NULL COMMENT '目的节点编码',
      `demands` int(11) DEFAULT '0' COMMENT '订单商品数量',
      PRIMARY KEY (`scheId`,`deliverCode`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='调度任务详表';

这张表中，我们记录调度任务的详细数据

#### 结果路线表

    CREATE TABLE `scheduling_route` (
      `scheId` int(11) NOT NULL COMMENT '调度任务编码',
      `routeId` int(11) NOT NULL COMMENT '路线编号',
      `totalDistance` int(11) NOT NULL COMMENT '路线总里程，单位：米',
      `deliverCode` varchar(1024) NOT NULL COMMENT '承接的发货单号，通过半角逗号间隔',
      PRIMARY KEY (`scheId`,`routeId`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='调度任务路线表';

这张表用于记录每一个调度任务对应的发货单、路线

    CREATE TABLE `scheduling_route_detail` (
      `scheId` int(11) NOT NULL COMMENT '调度任务编码',
      `routeId` int(11) NOT NULL COMMENT '路线编号',
      `nodeSeq` int(11) NOT NULL COMMENT '访问节点序号',
      `nodeId` varchar(32) NOT NULL COMMENT '节点编码',
      PRIMARY KEY (`scheId`,`routeId`,`nodeSeq`)
    ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='调度任务路线详表';

这张表记录每一条路线上的所有节点的顺序

## 示例代码

这里仅给出关键的实现代码以及伪代码

### 数据模型

首先，我们从数据模型开始，一步步的解决问题。

#### 距离矩阵

我们先来看最重要的部分——距离矩阵，or-tools的运算是一定需要距离矩阵的，因此我们先来考虑他的构建  
我们先将请求参数封装成Java对象

    public class ReqNode {
        @ApiModelProperty(value = "源节点编码",required = true)
        private String srcNodeId;
        @ApiModelProperty(value = "目的节点编码",required = true)
        private String dstNodeId;
        @ApiModelProperty(value = "发货单号",required = true)
        private String deliverCode;
        @ApiModelProperty(value = "商品数量",required = true)
        private Integer demands;
    }

我们在之前提到过，如果要实现任意的起点和终点，那么我们最终的矩阵是需要额外维护一个到所有距离都为0的虚拟仓库，也就是说我们的矩阵需要额外维护一行和一列  
首先我们看一下如何填充第一行和第一列

    //车辆路线最大值 = 单据数量
    int vehicleNumberMax = reqList.size();
    //节点数量
    int size = vehicleNumberMax * 2;
    //数组容量
    int arraySize = size + 1;
    //初始化路径矩阵
    int [][] routeArray = new int[arraySize][arraySize];
    //设置第一行
    Arrays.fill(routeArray[0], 0);
    //设置第一列
    for (int i = 0; i < routeArray.length; i++) {
        routeArray[i][0] = 0;
    }

在上述代码中，我们先拿出请求参数列表的长度，即一共有多少个发货单，我们之前有提到过，在or-tools中创建manager对象时`vehicleNumber`这个参数事实上是路线最大总数限制，因此这里我们将单据的数量作为最大限制即可，因为最差的情况下就是每个单据安排一条路线。  
由于我们的发货单中是包含起点和终点的，也就是说每张单中包含两个node点，这样我们乘以2就得到最终需要排线的点的数量，然后在此基础上加1，就得到了数组的容量。接下来我们就可以直接初始化二维数组矩阵，并且直接将第一行和第一列的值设置为0.

接下来，我们看一下如何对数组的其余数据进行设置

    //查出所有节点之间的距离,首先要拿出要用到的节点
    List<String> nodeList = new ArrayList<>();
    //真实节点对应的单号映射
    Multimap<String, String> nodeCodeMap = ArrayListMultimap.create();
    //这一波循环主要是为了拿出所有节点，构建顺序，并且构建容量矩阵，以及构建一个真实节点和单号的映射
    for (ReqNode reqNode : reqNodeList) {
        String srcNodeId = reqNode.getSrcNodeId();
        String deliverCode = reqNode.getDeliverCode();
        nodeCodeMap.put(srcNodeId,deliverCode);
        nodeList.add(srcNodeId);
        String dstNodeId = reqNode.getDstNodeId();
        nodeList.add(dstNodeId);
        nodeCodeMap.put(dstNodeId,deliverCode);
    }

首先我们需要将所有要排线的节点(所有的起点和终点)拿出来，放入一个新的List中，这里之所以使用for循环而不是steam，一是为了方便代码理解，二是为了方便做其他操作。  
接着，使用`Guava`包下面的`Multimap`来存放**真实节点** 对应的单号，这么做的原因暂时按下不表。`Multimap`的key重复的时候，他的value会自动扩充为一个List，这样方便处理一个key对应多个值的情况。我们在前面说到起点和终点是可以重复的，所以这里显然使用`Multimap`处理更加方便。  
这里可能会有疑问：一个起点和一个终点不是对应一张发货单号吗？为什么要起点和终点都放入map关联？  
在之后会给出解答。

> 进行初始处理后，处理矩阵数据

    //虚拟下标和真实node节点之间的映射
    Map<Integer,String> indexMap = new HashMap<>();
    //虚拟下标对应的单号的映射
    Map<Integer,String> indexCodeMap = new HashMap<>();
    //node节点对于虚拟下标的映射
    Multimap<String, Integer> nodeMap = ArrayListMultimap.create();
    //调用方法获取两两之间的距离
    Map<String,Integer> distanceMap = this.getDistanceMap(List<String> nodeList);
    //构建矩阵下标值，因为有占位0，所以下标从1开始
    //行
    int line = 1;
    for (String node1 : nodeList) {
        //列
        int column = 1;
        //取出node1-node2的距离并插入矩阵
        for (String node2 : nodeList) {
            String key = node1 + "-" + node2;
            Integer distance = distanceMap.get(key);
            routeArray[line][column] = distance;
            column++;
        }
        indexMap.put(line,node1);
        nodeMap.put(node1,line);
        List<String> codes = (List<String>) nodeCodeMap.get(node1);
        if (!CollectionUtils.isEmpty(codes)){
            indexCodeMap.put(line,codes.get(0));
            codes.remove(0);
        }
        line++;
    }

> 我们先来看一下`getDistanceMap`这个方法是如何计算两两之间的距离

    public Map<String, Integer> getDistanceMap(List<String> nodeIds) {
        Map<String,Integer> map = new HashMap<>();
        for (LogNode src : nodeIds) {
            String lng1 = src.getLongitude();
            String lat1 = src.getLatitude();
            String srcNodeId = src.getNodeId();
            for (LogNode dst : nodeIds) {
                String lng2 = dst.getLongitude();
                String lat2 = dst.getLatitude();
                int distance = DistanceUtils.getDistance(lng1, lat1, lng2, lat2);
                String dstNodeId = dst.getNodeId();
                String key = srcNodeId + "-" + dstNodeId;
                map.put(key,distance);
            }
        }
        return map;
    }

因为我们要计算node节点两两之间的距离，所以我们将`nodeIds`使用双层嵌套for循环，这样遍历后就相当于两两相交，然后最终map里面的key由"起点-终点"的形式存放，然后我们在外围的方法中取出就更方便，并且后期如果要修改距离计算逻辑，也只需要修改这里的代码，不用修改外围方法。  
这里两点之间的距离是依据经纬度来计算的直线距离，后续也可以通过百度地图或者Google地图的接口来计算导航距离。

> 我们看一下根据经纬度计算直线距离的代码

    public static int getDistance(String lng1, String lat1, String lng2, String lat2){
        // 使用正则表达式判断输入是否合法
        String regex = "^\\-?\\d+(\\.\\d+)?$";
        if (!lng1.matches(regex) || !lat1.matches(regex) || !lng2.matches(regex) || !lat2.matches(regex)) {
            return 0;
        }
        double radLat1 = Math.toRadians(Double.parseDouble(lat1));
    	double radLat2 = Math.toRadians(Double.parseDouble(lat2));
    	double a = radLat1 - radLat2;
    	double b = Math.toRadians(Double.parseDouble(lng1)) - Math.toRadians(Double.parseDouble(lng2));
    	double s = 2 * Math.asin(Math.sqrt(Math.pow(Math.sin(a/2), 2) + Math.cos(radLat1) * Math.cos(radLat2) * Math.pow(Math.sin(b/2), 2)));
    	return (int) Math.round(s * EARTH_RADIUS);
    }

> 回过头，继续看构建矩阵数据的代码

    //虚拟下标和真实node节点之间的映射
    Map<Integer,String> indexMap = new HashMap<>();
    //虚拟下标对应的单号的映射
    Map<Integer,String> indexCodeMap = new HashMap<>();
    //node节点对于虚拟下标的映射
    Multimap<String, Integer> nodeMap = ArrayListMultimap.create();

可以看到，这里定义了三个map：`indexMap`、`indexCodeMap`、`nodeMap`

  * indexMap ： key->虚拟下标 value->真实node节点 
    * 由于我们使用or-tools计算的时候最终的返回结果是虚拟下标，但最后要把虚拟下标解析成真实的节点存入数据库。因此就需要额外维护一个map，记录虚拟下标所对应的真实下标，方便我们后续进行处理。
    * 虚拟下标与真实节点是1对1的关系，因此使用普通的HashMap即可
  * indexCodeMap ： key-> 虚拟下标 value->发货单号 
    * 在前面，我们维护了一个真实节点对应的发货单号列表，但是我们在的真实节点是可能重复的，他会对应多个发货单，因此最后我们解析每条路线对应的发货单号的时候就要用到虚拟下标来获取(因为虚拟下标是唯一的)，这里也是1对1的关系。
  * nodeMap：key ->真实node节点 value->虚拟下标 
    * 真实节点对应虚拟下标的映射。这里是1对多的关系，故而使用`Multimap`

> 接着我们看一下具体如何将数据写入矩阵

    //构建矩阵下标值，因为有占位0，所以下标从1开始
    //行
    int line = 1;
    for (String node1 : nodeList) {
        //列
        int column = 1;
        //取出node1-node2的距离并插入矩阵
        for (String node2 : nodeList) {
            String key = node1 + "-" + node2;
            Integer distance = distanceMap.get(key);
            routeArray[line][column] = distance;
            column++;
        }
        indexMap.put(line,node1);
        nodeMap.put(node1,line);
        List<String> codes = (List<String>) nodeCodeMap.get(node1);
        if (!CollectionUtils.isEmpty(codes)){
            indexCodeMap.put(line,codes.get(0));
            codes.remove(0);
        }
        line++;
    }

外层循环表示本次要构建矩阵的node节点，内层循环用来计算他到其他所有节点的距离，换言之，外层是插入每个一维数组，内层是插入每个一维数组中的数据。  
在最外层，记录一个变量`line`代表行，也可以理解为二维数组中的一维数组，同时他也代表着每个真实节点对应的虚拟下标。  
然后在第一层的循环中记录一个变量`column`代表列，即每个一维数组中的元素。  
重点看一下我们如何将虚拟下标和发货单号进行关联

    List<String> codes = (List<String>) nodeCodeMap.get(node1);
    if (!CollectionUtils.isEmpty(codes)){
        indexCodeMap.put(line,codes.get(0));
        codes.remove(0);
    }

我们先拿到当前的node节点对应的所有发货单号，然后取出第一个元素，放入虚拟下标对应的映射map，这里就是利用了ArrayList的有序性，由于我们始终是用的`nodeList`这个列表，我们在添加`nodeCodeMap`的时候就是有序的，所以我们取出来也是有序的，然后每次取完就删除掉第一个元素，这样就完美的解决了node节点会有多个发货单号的问题。  
至此，我们完成了距离矩阵的构建。

#### 约束矩阵

约束矩阵是为了保证取送的顺序，显然每张发货单的起点到终点，这就是我们需要做的约束，而我们要考虑的就是将真实node节点的约束转换为虚拟下标。

    public int[][] getPickups(List<ReqNode> reqNodeList, Map<String, Collection<Integer>> map) {
        //防止原来的映射列表被破坏
        String jsonStr = JSONUtil.toJsonStr(map);
        Map<String, Collection<Integer>> nodeMap = JSONUtil.toBean(jsonStr, Map.class);
        //有几张单，就有几个约束
        int[][] pickupsDeliveries = new int[reqNodeList.size()][2];
        int flag = 0;
        for (ReqNode reqNode : reqNodeList) {
            String srcNodeId = reqNode.getSrcNodeId();
            String dstNodeId = reqNode.getDstNodeId();
            //起点为第一个元素
            setPickup(nodeMap, srcNodeId, pickupsDeliveries[flag],0);
            //终点为第二个元素
            setPickup(nodeMap, dstNodeId, pickupsDeliveries[flag],1);
            flag++;
        }
        return pickupsDeliveries;
    }

这段代码中，我们需要将接口的请求参数传入这个方法，并且需要传入真实node节点对应的虚拟下标的map  
这里看到，我将源数据转为json数据然后又转成了一个新的map，这主要是为了防止我们在取出的过程中破坏了原有的映射列表，如果你使用`map.putAll()`这个方法的话，由于他不是深拷贝，就会将原有列表破坏掉  
然后看一下取出映射的代码，和我们之前创建距离矩阵获取发货单号的原理是一样的，就是利用ArrayList他的有序性。

    public void setPickup(Map<String, Collection<Integer>> nodeMap, String nodeId, int[] pickupsDeliveries,int dst) {
        //一个节点会对应多个下标，按顺序取即可，取完移除掉
        List<Integer> indexList = (List<Integer>) nodeMap.get(nodeId);
        Integer index = indexList.get(0);
        pickupsDeliveries[dst] = index;
        indexList.remove(0);
    }

#### 

#### 车辆容载限制

这个就比较容易了，我们只需要设置与发货单数量一致的车辆容量矩阵，以及每个虚拟下标对应的商品数量就行了

##### 车辆容量

    long[] vehicleCapacities = new long[vehicleNumberMax];
    Arrays.fill(vehicleCapacities,vehicleCapacity);

##### 商品数量

我们修改第一次遍历请求参数列表的代码即可

    //默认0
    demands[0] = 0;
    int demandsIndex = 1;
    for (ReqNode reqNode : reqNodeList) {
        //起点的容量限制为输入值
        demands[demandsIndex] = reqNode.getDemands();
        demandsIndex++;
        //终点的容量限制为0
        demands[demandsIndex] = 0;
        demandsIndex++;
    }

这里要注意需要考虑到我们的虚拟仓库默认0，然后我们将起点的商品数量设置为单据发货数量，终点设置为0。  
当然可以根据实际的业务需求来调整。

### 解析结果

调用or-tools进行计算以及获取or-tools的计算结果的代码我们在第一篇已经给出了，在经过初始数据的处理后调用or-tools的方法即可，这里就不在赘述了。我们重点看一下如何将or-tools计算出的结果转换为真实的node节点，然后放入数据库  
先看一下我们在or-tools计算出结果的返回值的数据结构怎么设计：

    public class RspComputeRoutePlan {
        @ApiModelProperty("方案总里程")
        private Integer totalPlanDistance;
        @ApiModelProperty("方案路线")
        private List<RspComputeRoute> resRouteList
    }

这个对象用来记录该调度方案的总里程，以及包含的所有路线

    public class RspComputeRoute {
        @ApiModelProperty("路线总里程")
        private Integer routeDistance;
        @ApiModelProperty("路径：以,来分隔每个点的顺序")
        private String route;
    }

每一条路线上的所有点的顺序我们用","来分隔，这样方便我们后面解析  
这里我就不给出具体的计算过程的代码了，接下来我们看下拿到这个计算结果后怎么解析并放入数据库

> 最终需要的数据结构

    public class RspSmartPlan {
        @ApiModelProperty("路线列表")
        List<RspSmartPlanRoute> routeList;
    
        @ApiModelProperty("方案总里程")
        private Integer totalPlanDistance;
    
    }

    public class RspSmartPlanRoute {
    
        @ApiModelProperty("路线编号")
        private Integer routeId;
    
        @ApiModelProperty("任务编号")
        private Integer scheId;
    
        @ApiModelProperty("路线总距离")
        private Integer routeDistance;
    
        @ApiModelProperty("承接的发货单号，通过半角逗号间隔")
        private String deliverCode;
    
        @ApiModelProperty("节点序号路径")
        private List<RspSmartRouteNode> routeList;
    
    }

    public class RspSmartRouteNode {
        @ApiModelProperty("节点编号")
        private String nodeId;
        @ApiModelProperty("节点序号")
        private Integer nodeSeq;
        @ApiModelProperty("经度")
        private String longitude;
        @ApiModelProperty("纬度")
        private String latitude;
        @ApiModelProperty("名称")
        private String name;
    }

> 解析结果

我们最终解析结果必备的参数：

  * `indexMap`：虚拟下标对应真实node节点的映射，这个之前我们已经维护过了，这里拿来直接用即可
  * `bestPlan`: 我们从or-tools中计算出来的结果集
  * `indexCodeMap`: 虚拟下标对应的发货单
  * `scheId`: 当前的调度任务，这里由于我们计算是异步任务，所以由前置方法传过来就行

接下来看看具体代码：

    private static List<RspSmartPlanRoute> getRspSmartPlanRoutes(Map<Integer, String> indexMap, RspComputeRoutePlan bestPlan, Map<Integer, String> indexCodeMap, Integer scheId) {
        //返回路线结果集
        List<RspSmartPlanRoute> rspList = new ArrayList<>();
        //计算出来的路线列表
        List<RspComputeRoute> resRouteList = bestPlan.getResRouteList();
        //路线编号，自增
        Integer routeId = 1;
        for (RspComputeRoute rspComputeRoute : resRouteList) {
            //获取路径
            String route = rspComputeRoute.getRoute();
            //空跑不做记录
            if (StringUtils.isEmpty(route)) continue;
            //转为list
            List<Integer> indexList = Arrays.stream(route.split(",")).map(Integer::parseInt).collect(Collectors.toList());
            //发货单列表
            Set<String> deliveryCodeSet = new HashSet<>();
            //找到发货单列表
            for (Integer index : indexList) {
                String s = indexCodeMap.get(index);
                if (StringUtils.isNotEmpty(s)) deliveryCodeSet.add(s);
            }
            //发货单列表转为一个String
            String deliveryCodes = String.join(",",deliveryCodeSet);
            //转为真实的nodeId列表
            List<String> nodeList = indexList.stream().map(index -> {
                String nodeId = indexMap.get(index);
                return nodeId;
            }).distinct().collect(Collectors.toList());
            //返回对象
            RspSmartPlanRoute rspSmartPlanRoute = new RspSmartPlanRoute();
            //节点路径及下标列表
            List<RspSmartRouteNode> routeList = new ArrayList<>();
            for (int i = 1; i <= nodeList.size(); i++) {
                String nodeId = nodeList.get(i - 1);
                RspSmartRouteNode rspSmartRouteNode = new RspSmartRouteNode();
                rspSmartRouteNode.setNodeId(nodeId);
                rspSmartRouteNode.setNodeSeq(i);
                routeList.add(rspSmartRouteNode);
            }
            rspSmartPlanRoute.setRouteId(routeId);
            rspSmartPlanRoute.setDeliverCode(deliveryCodes);
            rspSmartPlanRoute.setRouteDistance(rspComputeRoute.getRouteDistance());
            rspSmartPlanRoute.setRouteList(routeList);
            rspList.add(rspSmartPlanRoute);
            routeId++;
        }
        return rspList;
    }

代码比较长，我们来一段一段的看。  
首先定义一个变量`routeId`，用来记录每条路线的索引顺序  
接着我们取出刚才从or-tools中获取的计算结果路径列表`resRouteList`，并进行一次for循环，处理每一条路径

> 很明显，我们的目标如下：

  * 将虚拟下标转换为真实node节点
  * 找出每条路线对应的发货单号

我们看一下for循环中的这段代码：

    //获取路径
    String route = rspComputeRoute.getRoute();
    //空跑不做记录
    if (StringUtils.isEmpty(route)) continue;
    //转为list
    List<Integer> indexList = Arrays.stream(route.split(",")).map(Integer::parseInt).collect(Collectors.toList());
    //发货单列表
    Set<String> deliveryCodeSet = new HashSet<>();
    //找到发货单列表
    for (Integer index : indexList) {
        String s = indexCodeMap.get(index);
        if (StringUtils.isNotEmpty(s)) deliveryCodeSet.add(s);
    }
    //发货单列表转为一个String
    String deliveryCodes = String.join(",",deliveryCodeSet);

我们之前有提到过，我们在进行运算的时候，传入的`vehicleNumberMax`这个参数实际上指定的是**最大路线容量限制** ，如果有多余的路线，距离为0，并且路径点为空，所以这里如果取出路径为空，可以直接跳过这条数据的循环。当然你也可以在使用or-tools计算结果的时候就将空跑的路径删除。  
我们的路径是由","来分隔的，直接用Stream转成一个List`indexList`，接下来利用`Set`存储这条路径对应的发货单号。然后我们对这个`indexList`进行遍历，通过`indexCodeMap`找出他对应的发货单号，这里不用区分起点和终点，因为我们是用`Set`存放发货单号的，可以自动去重。  
如果你使用“起点-终点”作为key的方式存放map的话，那就要使用双层嵌套循环，大大的影响了性能，所以直接不论起点终点直接往里面添加，自动去重是最好的解决方法。  
最后将发货单列表转换为一个字符串，方便我们放入数据库存储。

> 将虚拟节点解析成真实节点

    //转为真实的nodeId列表
    List<String> nodeList = indexList.stream().map(index -> {
        String nodeId = indexMap.get(index);
        return nodeId;
    }).distinct().collect(Collectors.toList());
    //返回对象
    RspSmartPlanRoute rspSmartPlanRoute = new RspSmartPlanRoute();
    //节点路径及下标列表
    List<RspSmartRouteNode> routeList = new ArrayList<>();
    for (int i = 1; i <= nodeList.size(); i++) {
        String nodeId = nodeList.get(i - 1);
        RspSmartRouteNode rspSmartRouteNode = new RspSmartRouteNode();
        rspSmartRouteNode.setNodeId(nodeId);
        rspSmartRouteNode.setNodeSeq(i);
        routeList.add(rspSmartRouteNode);
    }

这段代码，没什么好讲的，由于多个虚拟下标可能会同时对应一个node节点，注意去重即可

## 总结

解决这个问题的顺序无非就是先构建参数传入or-tools，然后解析结果。关键我们要把握虚拟下标和真实节点之间的转换关系。  
这篇文章解决了一个应用示例，在下一期中，我们将继续优化车辆容量限制的代码，可以根据不同的车型容量，推荐出不同的车辆类型组合。

---

![sensen](images/sensen.png)
