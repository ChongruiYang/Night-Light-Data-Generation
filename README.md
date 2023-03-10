# VIIRS-DNB_analysis

先说结论:

1) 我们已经获取到由NOAA提供，经CSM处理的2019年1月至2022年9月由VIIRS卫星传输回的月度灯光数据，并通过异常值处理与统计将2019，2020，2021三年冬季的灯光栅格转换成了地区范围的月度灯光强度, 其对照数据（201903，201904）的结果与现有对照的两个数据源基本一致（分别是GNLD数据库，中国矿业大学课题组；最新月份的数据其他两个数据源并未公布，因此对比2019年3月与4月数据）
2) 我们认为对月度灯光数据制作长期可比的时间序列并不现实，其原因是夏季北半球的杂散光被去除后会导致部分区域夏季无法被观测（且某些地区的夏季降雨较多影响观测日期），具体而言:5-9月份北方都有一部分区域难以被卫星系统性观测；但如果单独分析北半球秋冬季的灯光数据，用其对比应该是较为现实的（但研究假设，疫情封城后，去除季节性因素后，临近月份灯光亮度降低的假设不一定成立）
3) 由于VIIRS传感器较新且没有更换过，基本不存在因为传感器校正或者老化而导致的数据失真问题
4) 主要需要讨论的问题有：1' 可能的秋冬季月度灯光数据的用途，若选取秋冬季数据是否能满足我们的需求 2' 如何构造灯光指标，是行政区划平均值，中位数，某个分位数还是建成区的平均值
5) 查看一些额外的假设验证与分析, 例如武汉在2019年12月至2020年1月光照强度确有下降，但不清楚这种变化是否确实受到疫情的影响


# 如何处理SNPP-VIIRS的灯光数据

1. 从官方数据接口提供的数据下载灯光数据集，一般为tiff或者geotiff格式；并下载最新的中国省市县区级行政边界1：400 shapefile文件(一定要是最新的，不然某些海岸线会发生变化）

2. 在Arcgis Pro中，添加数据 -- 打开图层，得到一个栅格数据文件（灯光数据），一个要素图层文件（城市边界）。然后对SNPP-VIIRS的栅格要素裁剪到城市图层上来，并及时保存裁剪的栅格数据。

    1） 在工具栏中搜索 -- 裁剪栅格，输入SNPP-VIIRS光源地图，依据地级市城市的边界进行裁剪（√）
    
    2） 右击已经创建完毕的图层 -- 导出栅格，选择合适的地理边界对它进行导出(当然也可以不选择这么做)


3. 调整投影坐标系与重采样。在工具栏里输入 -- “投影” 或 “投影栅格”，选中灯光栅格数据，将原先的图层投影到WGS 1984上（√）
   
   1） 如果X轴与y轴的像元不为整数，可以同时调整x与y像元的大小，例如将416改为500，方法选择“最近邻法”便于计算（当然也可以不选择这么做）

   2） Arcgis Pro使用的世界地形底图投影坐标系为WGS 1984 Web Mercator (auxiliary sphere)，EPSG:3587；而Payne Institute提供的无云DNB数据集的坐标轴投影参考为WGS 1984，EPSG:4326；中国的区县边界地图坐标轴投影参考多为WGS 1984 Asia_North_Albers_Equal_Area_Conic，EPSG:102025。在不需要使用底图做其他地理统计分析的情况下，实际上只需要保证地级市边界与灯光数据集的范围一致即可(例如WGS 1984)。
   
   3） 右击图层的“属性”选项卡，查看投影参考坐标与xy轴像素大小（√）
   
   4）如果需要对像元进行四舍五入，或者对像元值进行缩放分级，需要在工具箱 -- 重采样，划定采样的分级与方法（因为DMSP的图像为64级，如需要将SNPP-VIIRS与DMSP进行匹配，则可能需要转换标度）
   
4. 异常值处理。需要注意的是这一步仅能识别一些统计范围内可能存在的信源异常值，例如<0，过大的局部值等；而卫星型号，数据可比性等问题不会在这一步得到解决。可选择的办法有：

   1） 直接对数据在区域内以像元为单位进行截断，统计每一个地理区块内部的所有像元值异常分位数，截取1% - 99%分位数内的数据（X）
     
   2） 参考GLND数据库对SNPP-VIIRS数据的处理方法，提取当前时间段"北京","上海"，"广州"范围内的像元最大值为全国的"有效最大值"。利用栅格计算工具将全图的像元最大值替换为"有效最大值"（√）
   
   3） 提取当前时间段"北京","上海"，"广州"范围内的像元最大值为全国的"有效最大值"。当某个像元值大于"有效最大值"时，以周围的八个像元的平均值替代该像元值。或者对所有的像元进行"低通滤波"（X）
   
   4） 由于Payne Institute提供的处理产品已经消除了杂散光的影响，且我们并未有与DMSP相互校正的需求，这里并未与DMSP做互相的掩膜提取比较<0值的区域是否真正<0。而是直接参照一些文献的方法，将<0的像元值视为异常值全部替换为0 （√）
   
   5）异常值掩膜处理。对月度数据的修正可以获取该年度的各种类型的lit mask，用来剔除掉短时间存在的异常光源。换句话说，如果一个像素在2013年的月度记录中被记录为有光，但在2015年的年度数据中NOAA的科学家们应用他们的算法清理后，却显示为无光区域。那么可以将其视为异常值并剔除掉。其做法是载入掩膜栅格数据与灯光图层，依据栅格表达式将年度数据掩膜为0的月度像元赋值为0.（X）

     

5.  对栅格的pixel value由浮点型转换成整型（pixel value不是整型在栅格的拼接上会出现问题，但与要素的连接不会，这一步可以不做）：

    1） 打开Arcgis Pro的工具箱 -- “栅格计算器”，在命令行输入计算表达式：
      `Con((RoundUp("NPP_VIIRS_202011.tif")- 
      "NPP_VIIRS_202011.tif")>0.5,RoundDown("NPP_VIIRS_202011.tif"),RoundUp("NPP_VIIRS_202011.tif"))`，这是做四舍五入的操作。当然如果pixel value过小，则应当先全部放大乘于整数后再转换成整型，在空间分区统计之后再转换成原先的大小(在输入栅格计算表达式时一定要注意空格与符号的半角与全角不然可能无法运行）

    2） 在工具栏里输入 -- “转为整型”并搜索，导入经过四舍五入转换过后的整数浮点值，可以直接转换成整型
     
6. 在工具栏里输入 -- “以表格显示分区统计”，将行政边界文件输入“要素区域数据”，将栅格文件输入“赋值栅格”，计算得到一张以地方区县界为统计单元的表，其中会提供一系列的统计信息。`count`代表落在该区域内的像元个数，`area`代表区域面积，`sum`代表区域内的像元值加总，`mean`代表区域的像元值加总除以总面积。

# 示例处理过程 -- SNPP-VIIRS 201903

1）导入由Clorado Payne Institute下载的原始数据文件`SVDNB_npp_20190301-20190331_75N060E_vcmcfg_v10_c201904071900.avg_rade9h.tif`, 其显示东亚半区的所有图像，像元值范围从-0.82到48754.1，"avg_rade9h" 代表其显示的是平均辐射度四舍五入到百分位

2）依据上述方法与中国的省市边界裁剪后得到中国全域灯光图，像元值范围从-0.57到2869.97

3）先做一次"以表格显示分区统计"以获得基本统计数据，2019年3月的北京最大亮度栅格为271.890015，上海最大亮度栅格为284.109985，广州为306.600006，那么将全地图像元值大于306.6的像元值全部替换成306.6--`Con(("SVDNB_npp_2019030120190_Clip1">306.6),306.6,"SVDNB_npp_2019030120190_Clip1")`。同时将小于0的像元值全部替换为0-- `Con(("con_raste"<0),0,"con_raste")`. 我们得到一个像元值在0-306.6的中国灯光地图。

<div align=center><image src= "https://user-images.githubusercontent.com/82168423/215340056-edd644e4-ebef-4dab-acd9-42bcb20e0170.png" /></div>
<div align=center>Table 1. VIIRS-DNB</div>

   同时我们分别展示像元值大于0.5，像元值大于1，像元值大于5的掩膜地图(以0与1表示数值范围，顺序由上到下)

<div align=center><image src="https://user-images.githubusercontent.com/82168423/215341989-34a647c4-c54c-4b9e-8b1b-2bdb9a78be33.png" /></div>
<div align=center>Table 2. PIXEL over 0.5</div>
   
<div align=center><image src="https://user-images.githubusercontent.com/82168423/215342040-959e1431-f6e6-4e63-a1a2-c634ed26beb1.png" /></div>
<div align=center>Table 3. PIXEL over 1</div>

<div align=center><image src="https://user-images.githubusercontent.com/82168423/215342147-57758ab1-4b1a-4409-85b1-5dff0fc916c1.png" /></div>
<div align=center>Table 4. PIXEL over 5</div>


4）再次运行"以表格显示分区统计"(只计算有像元值地区),得到数据集201903_light.csv, 一般选用其中的`mean`作为`DNvalue`的替代指标


# Robust Check of Generate Way
1. 由于对原始数据集不同的处理方法导致灯光pixel的标度不一，我们同时寻找一些国内外地理测绘使用的经过转换后的SNPP-VIIRS数据集，观察归一化后中国各个主要城市的DNvalue的差异情况，如果各份数据集之间的差异并不大，则说明其处理效果良好
   
   1) 201903与201904的灯光数据，与GNLD处理数据集的结果相比，整体上有baseline 像元值被加1了的情况，数据内部的峰度与偏度并没有太大改变，推测是他们对负值进行掩膜处理的时候直接加上了1使之变为正数，但只要相对标度一致平移变换并不会产生太大问题
   
   2) 201903与201904的灯光数据，与[中国矿业大学课题组](http://www.geodoi.ac.cn/WebCn/doi.aspx?Id=3100)提供的处理后数据完全一致
   
2. 存在的问题:

   1) 月度数据与年度数据不同，存在"像元值的季节差异性": 夏季全国大部分地区的像元值通常要小于冬季，不论是干旱地区还是南方地区（去除夏季0值之后差距减小），可能与阴雨天气（例如四川盆地）和生产活动有关。且由于卫星观测原因，夏季部分月的光照情况并不好。是否需要用到数据质量不好的这几个月的数据，或是需要插值？(NOAA在夏季的光源检测偏低，特别是热带或由于太阳照射造成的，因而在平均辐射图像中数值为零的并不意味该地区数值为零，也有可能是没有观察到灯光值。而出现DN均值为0的一般是每年的6月份和7月份。这可能是因为中国进入夏季，部分地域无法监测到数据，但不代表该地区DN均值为0。每月夜间灯光的全球覆盖时间受一年中不同时间白天长短的影响很大。在夏季，由于白天较长，北半球的夜间覆盖较少）。我们检查CSM提供的数据时也发现了每年某些月份，高纬度地区的灯光数据容易出现大面积缺失，这主要是对卫星数据形成的杂散光的处理造成的。
  
   2) 
   <table align=center>
    <tr>
        <th><image src="https://user-images.githubusercontent.com/82168423/215971999-b017ce32-afe4-4429-b11c-7a3ae902935f.png" height="400" width="400"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/215979100-2e96e9ad-2e56-490c-a170-e74d7b3f9eee.png" height="400" width="400"/></th>
    </tr>
    <tr>
        <th>Table 5. June, 2022, Data from CSM (Green Pixel over 0.2)</th>
        <th>Table 6. July, 2022, Data from CSM (Green Pixel over 0.2)</th>
    </tr>
    <tr>
        <th><image src="https://user-images.githubusercontent.com/82168423/215989207-338dec6e-2d53-4ac0-9752-5ee4fd496377.png" height="400" width="400"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/215987405-5d90d2ff-9d17-4627-b41d-e4c0b2b4d816.png" height="400" width="400"/></th>
    </tr>
    <tr>
        <th>Table 7. Jan, 2022, Data from CSM (Green Pixel over 0.2)</th>
        <th>Table 8. Jan, 2022, Data from CSM (Cover of No cloud Days, white means more days no cloud)</th>
    </tr>
</table>

   3) 以下是2021年各个月度CMS提供的无云观测日期个数的统计图，我们很容易发现无云日期与无法观测日期在全年内的规律（值得注意的是，无法观测的界限实际上是一条淡淡的弧线而不是恰好落在某个纬度上,这个纬度仅仅是一个大致肉眼观测的纬度，具体未测度线可以通过栅格计算器计算每个区域的位置），我们可以观测到哪些数据是可以使用的

   <table align=center>
    <tr>
        <th><image src="https://user-images.githubusercontent.com/82168423/216002527-7bbb4a36-4785-4d10-aab2-42f5f9269f3c.png" height="200" width="200"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/216002584-7ff54cfa-6224-4c12-9634-fdfc95c0aa6f.png" height="200" width="200"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/216012649-f3dd1da6-899b-4ca3-804e-0f6059569d53.png" height="200" width="200"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/216070600-1981ca0f-e612-4745-890f-63d8b75e5f49.png" height="200" width="200"/></th>
    </tr>
    <tr>
        <th>Jan, 2021</th>
        <th>Feb, 2021</th>
        <th>Mar, 2021</th>
        <th>Apr, 2021(50'N unobserved)</th>
    </tr>
    <tr>
        <th><image src= "https://user-images.githubusercontent.com/82168423/216071849-418db03f-13b8-454d-8b51-4a6375a7fa7c.png" height="200" width="200"/></th>
        <th><image src= "https://user-images.githubusercontent.com/82168423/216072226-b27f2682-b15a-46c2-b361-cb7ce2d8baca.png" height="200" width="200"/></th>
        <th><image src= "https://user-images.githubusercontent.com/82168423/216072627-d1b57a27-726b-4e20-b4f7-04ffa4d7bba4.png" height="200" width="200"/></th>
        <th><image src= "https://user-images.githubusercontent.com/82168423/216073310-126d581a-3860-453b-9adc-e883f35e58fb.png" height="200" width="200"/></th>
    </tr>
    <tr>
        <th>May, 2021(38'N unobserved)</th>
        <th>Jun, 2021(33'N unobserved)</th>
        <th>Jul, 2021(35'N unobserved)</th>
        <th>Aug, 2021(40'N unobserved)</th>
    </tr>
    <tr>
        <th><image src="https://user-images.githubusercontent.com/82168423/216073771-992b70ee-bf1a-404d-bb39-7dbd19191725.png" height="200" width="200"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/216074219-51454842-07e6-40a4-9cee-0b6146bd4919.png" height="200" width="200"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/216074824-cb2ba200-f824-4435-b92b-22548ac8e666.png" height="200" width="200"/></th>
        <th><image src="https://user-images.githubusercontent.com/82168423/216002584-7ff54cfa-6224-4c12-9634-fdfc95c0aa6f.png" height="200" width="200"/></th>
    </tr>
    <tr>
        <th>Sep, 2021(50'N unobserved)</th>
        <th>Oct, 2021</th>
        <th>Nov, 2021</th>
        <th>Dem, 2021</th>
    </tr> 
</table>

   4) 如何选择灯光强度的代理指标，是平均数，中位数，分位数还是其他值。某些地区的灯光值不一定完全对应经济活动，有记录的大部分时间里，天津的灯光均值以及中位数都大于北京，但不能说明天津的经济活动总体来说高于北京，可能是城市总面积的原因，北京的像元个数大致有99000且西北多山，有很多未能采集到像素的像元格，天津的像元个数大致有72000（在做灯光处理时，是否在平均处理的时候区分"建成区面积"与”总面积"）
 
      4.1) 秦蒙、刘修岩（2015）、刘修岩等（2016）利用DSMP/OLS夜间灯光数据对中国的城市区域范围进行了界定。具体的方法是，借助全球夜间灯光数据，用Arcgis软件提取出以灯光阈值作为判断标准的城市区域轮廓，然后再依据已判定的粗略城市区域，基于LandScan全球人口动态统计分析数据进一步提取同时满足灯光和人口密度阈值标准的、较为精确的城市区域。这种方法结合灯光亮度和人口密度来确定城市空间范围，较大程度地避免人口统计偏误和灯光溢出的干扰，比较准确、细致地刻画了真实的城市轮廓和面积。
  
   5) "现有的文献表明，灯光数据其本身的时间序列预测特性就较差，且这些因素更多是系统性问题引起的"。当前大部分研究在时间序列尺度上均采用年度数据，而区域的年度灯光数据往往基于全年中有观测的月份生成，这在年度上可行但在月度上失去了连续性；我们需要明确灯光数据具体的用途是什么，如果是观测冬季“封城”，“解封”等区间是否出现灯光亮度变化，采用“去季节性统计方法（例如算季节性指数）”调整后，我们认为额外扰动与气象因素对于灯光数据质量的影响将较小:
      
      5.1) [(Xi Chen and William D. Nordhaus;2019)](https://www.mdpi.com/2072-4292/11/9/1057)的结果表明，夜间灯光与GDP的相关性在MSA层面（美国的大都会统计区，多为城市建成区）上比在州层面上更强，这可能有两个原因。首先，MSA的GDP估计误差可能较小，而农业、渔业、林业和采矿业(高度集中在农村地区)的估计误差可能较大。因此，在MSA水平上，相关性可能会表现得更强。同时，主要发生在MSA的经济活动类型，如服务、零售、运输等，更有可能被夜间灯光捕捉到，而非MSA地区的经济活动主要是在不太依赖电力的行业，如农业、采矿或林业

      5.2) [(John Gibson et.al;2021)](https://www.sciencedirect.com/science/article/abs/pii/S0304387820301772)也认为夜间灯光和其他遥感数据不能很好地预测活动的时间序列变化，即使它们可以很好地预测横断面上的经济变量

      5.3) [(John Gibson;2020)](https://www.econstor.eu/bitstream/10419/230506/1/1688648216.pdf) 也与我们对SNPP-VIIRS数据有类似的观察。Clorado Payne Institute 提供的SNPP-VIIRS月度数据，剔除了受杂散光和云层影响的像素点，剩下的数据仍然受到极光和气体耀斑等短暂光源的影响。为了提高月度数据与年度数据可比性，其建议删除记录为无永久记录的任何像素的观测数据，也就是以年度数据作为掩膜处理月度数据。但这样做的一个问题是，对于典型的欧洲中纬度地区来说，部分时期的VIIRS文件没有记录灯光，因为在漫长的夏季夜晚，杂散光的影响已经被过滤掉了。有一个明显的南北模式，较北的地区只有6个月有数据(1月至3月和10月至12月）；南欧地区有9个月或更多的数据。Gibson创建的VIIRS年度估计只使用1月至3月和10月至12月的月度数据

<div align=center><image src= "https://user-images.githubusercontent.com/82168423/216259374-db835c0e-4d38-4b42-b229-bdc42063411d.png" /></div>
<div align=center>Months Can be Observed in EUROPE</div>

  5.4) [(Davin Chor&Bingjing Li;2021)](https://www.nber.org/system/files/working_papers/w29349/w29349.pdf) 同样使用SNPP-VIIRS月度数据对中美关税贸易进行了分析。他们不但观察了整个城市的变化，也同时观察了苏州城区，工业区，新技术开发区等不同城市内部细节的特点:从2018年第一季度到2019年第一季度，整个城市的夜间灯光都变暗了，但城市内部有很大的变化。苏州高新区、工业园区和其他地区的平均原木夜灯同比变化分别为-0.105、-0.085和-0.067。在出口敞口较大的工业区，夜间照明的降幅较大，这表明关税与对当地经济活动的不利影响有关
      
他们使用的原灯光数据为15 arc sec per pixel（482m * 482m），并将其重分类到0.1 arc degree per pixel（11KM * 11KM）的区域，这样全国仅包含97313个栅格（我们的数据中全国则包含55400827个栅格，不含港澳台），这样做的原因是他们将公司级别的数据匹配回地理栅格，希望探究栅格级别的变化，但原始的栅格数据范围太小。同时，他们选用了两个版本的数据，一是抹去所有可能造成杂散光影响的数据（即我们相同的数据源，可用的月份并不多），另一个是经过"杂散光校正程序"的，用作稳健性检验，其夏季的覆盖范围更大，但相对的，数据质量并不如完全没有杂散光的数据质量高。并且他们计算每一个地区不同年度的光强的增量变化，并在每一年中对1%与99%以上的值进行缩尾处理。他们并未对夏季的系统性灯光的bias进行处理，因为他们的数据使用计算的值是月度同比，即该年的灯光亮度与上一年该月的灯光亮度之间的增量，若该月份一直系统性观测不到，则其变化量一直为0

























     
- 2023/2/5 第一版数据处理

1) 拿到CMS提供的数据之后，校正坐标系与地图（source：https://github.com/GaryBikini/ChinaAdminDivisonSHP）到WGS 1984
2) 采用经济强度法，选择“北京”，“上海”，“广州”三个城市范围内的最大灯光栅格单元值，以此作为阈值，将全国其他地方高于此阈值的栅格替换
3) 统计每个城市地理边界内的灯光强度指标，count代表该城市内的像元个数，MEAN代表平均值，即我们想用的简单平均值


需要注意的地方：提供的城市边界包括：直辖市，地级市与省（建设兵团）直接管理的县级市，以及港澳台，若不需要可以删去。重庆由于地理范围太大，在原始地图中被分为了重庆城区（市辖区）与重庆郊县（县，自治县），若需要全市平均，可使用重庆城区值与重庆郊县平均值乘于各自的count再除掉总数。且2022年8月当月数据来自于卫星NOAA-20(https://eogdata.mines.edu/products/vnl/#rendering)。具体情况请参见CSM数据提供官网，仍然推荐使用10月份-次年3月份的数据，夏季数据由于阴雨天与光散射质量并不高。

由于老师担心我对低通滤波理解不够，我仍然使用了经济强度法做剔除，以下是每个时间节点上全图的最大值分布；最小值均为0. 在全国范围内，我们的数据包含55400827个栅格，不含港澳台。

   
- 变量说明：
ct_name:城市名称

ZONE_CODE：区域地理代码（仅作为标识符）

COUNT：在行政边界内的像元个数

AREA：行政边界内部的像元覆盖面积（需要注意的是这样计算出来的平方千米面积与城市范围的实际平方千米面积有一点差异，这是由于1：400地图的分辨率精度与像元方形大小-行政边界匹配的差异，但这与最终计算的灯光统计值无关，因为统计值基于的是落在地理区域内的像元个数，并不涉及到面积的计算；且落在边界上的点并不多）

MIN：区域内的像元最小值

MAX：区域内的像元最大值

RANGE：区域内像元的离差

MEAN：区域内像元的平均值

STD：区域内像元的标准差

SUM：区域内像元的总值

MEDIAN：区域内像元的中位数	

PCT90：区域内像元的90分位数

time：时间（年-月）


   - 2023/2/10 第二版数据处理

1）同样使用CSM处理过的月度灯光数据，其过滤了杂散光的影响，但会导致夏季北部地区缺少观测值。

2）采用2019年为基期，选取2019_composite_average_mask文件作为掩膜（这一数据由CSM提供，是他们合成的年度平均数据），提取出其中每个行政区域中大于像元值1%，5%，10%，20%，50%的像元值作为建成区域。并在确定建成区域后沿用这个建成区到之后的每一个月度的灯光数据上，计算之前所述的相同统计量。以下是各种不同分位数的建成区掩膜，由于1%，5%，10%的掩膜都非常相近（因为大部分地区的1%，5%，10%的像元值都是0），故仅制作10%，20%，50%以上区域的掩膜。观察地图我们可以知道改变不同分位数导致建成区的缩小主要集中在东部城市。

<div align=center><image src= "https://user-images.githubusercontent.com/82168423/218945027-9123d125-22cc-48d2-a4ba-62fda08442e6.png" /></div>
<div align=center>2019 year average light over 10% areas of each city (Mask)</div>

<div align=center><image src= "https://user-images.githubusercontent.com/82168423/218945598-e6f0b60f-9fc9-409f-9751-00bbe0309f40.png" /></div>
<div align=center>2019 year average light over 20% areas of each city (Mask)</div>

<div align=center><image src= "https://user-images.githubusercontent.com/82168423/218946163-39caa07a-4ce5-45aa-8905-78221c3d5f70.png" /></div>
<div align=center>2019 year average light over 50% areas of each city (Mask)</div>


参见Arcgis Pro功能：分区统计与栅格表达式：`Con(( "VNL_v21_npp_2019_global_Clip"> "avg_percent1"), 1,0)`,`SetNull("avg_1_cover"==0,"SVDNB_npp_20190101-20190131_75N060E_vcmcfg_v10_c201905201300.avg_rade9h.tif")`，这一步将掩膜不超过分位数的区域替换成无观测值。

3）对所有城市中每个月可能存在的异常值做截断，仍然采用经济强度法，并对北京，上海，广州三地最大值超过1000的月份替换成1000（正常的月度像元值应该在500左右），以避免像元单位内的光源值过大，每月最大值的threshold可以见表：[max_pixel.csv](https://github.com/JRCCE/VIIRS-DNB_analysis/files/10716385/max_pixel.csv)

4）分区域统计所在地区的像元平均值与分位数

5）合并历年的像元统计值文件，并检查每年每月的每一个文件中的统计像元格个数是否一致

注意：
1）对于5-8月份的数据，由于杂散光原因，观测非常不可信。例如北京的2020年5月份光照值，在10%的版本中仅为0.6，像元加总观测值仅为30000。在查询天气预报确认该年5月份基本无雨后，检查CSM提供的无云观测记录，发现此时北京的可观测日期多为1日或0日，天津同月份观测情况稍好但也存在这一现象。这是系统性的误差，所以最好不要使用5-8月份的数据，而全国数据可以使用对比的为10月份至次年3月份数据。再重复一下：不是有数据的地方就能用，看每个区域的summary！

处理后数据：

see：[light_over_20.xlsx](https://github.com/JRCCE/VIIRS-DNB_analysis/files/10740888/light_over_20.xlsx)；[light_over_50.xlsx](https://github.com/JRCCE/VIIRS-DNB_analysis/files/10740895/light_over_50.xlsx)；[light_over_10.xlsx](https://github.com/JRCCE/VIIRS-DNB_analysis/files/10740904/light_over_10.xlsx)
