<span style="color:yellow;font-size:27px;font-family:楷体;">Tip: 导出Excel需要设置响应头(规范)</span>

# 通用导入excel模板

### 分析表头

```java
@Slf4j
public class CommonReadExcelListener<M extends BaseMapper<T>,T> extends AnalysisEventListener<T> {

    private final static int BATCH_NUM = 1000;
    private final int batchNum;
    private int rows;
    private boolean validateSwitch = true;
    List<T> data = new ArrayList<>();

    private final IService<T> service;


    /*
     * 用于校验excel模板正确性的字段
     * */
    private final Field[] fields;
    private final Class<T> clazz;

    public CommonReadExcelListener(Class clazz,ServiceImpl<M,T> service){
        this(clazz,service,BATCH_NUM);
    }

    public CommonReadExcelListener(Class clazz,ServiceImpl<M,T> service,int batchNum){
        this.clazz = clazz;
        this.fields = clazz.getDeclaredFields();
        this.service = service;
        this.batchNum = batchNum;
    }

    /**
     * 每解析一行数据就调用该方法
     * @param t 实体
     * @param analysisContext 分析上下文
     */
    @Override
    public void invoke(T t, AnalysisContext analysisContext) {
        data.add(t);
        if (data.size() >= batchNum){
            service.saveBatch(data,batchNum);
            rows += data.size();
            data.clear();
        }
    }

    /**
     * 每解析完一页sheet页就会调用该方法
     * @param analysisContext 分析上下文
     */
    @Override
    public void doAfterAllAnalysed(AnalysisContext analysisContext) {
        log.info("读取当前文件第【"+(analysisContext.readSheetHolder().getSheetNo() + 1)+"】sheet页");
        if (data.size() > 0){
            service.saveBatch(data,batchNum);
            rows += data.size();
        }
        data = new ArrayList<>();
    }

    /*
     * 读取到excel头信息时触发，会将表头数据转为Map集合（用于校验导入的excel文件与模板是否匹配）
     *   注意点1：当前校验逻辑不适用于多行头模板（如果是多行头的文件，请关闭表头验证）；
     *   注意点2：使用当前监听器的导入场景，模型类不允许出现既未忽略、又未使用ExcelProperty注解的字段；
     * */
    @Override
    public void invokeHeadMap(Map<Integer, String> headMap, AnalysisContext context) {
        if (validateSwitch) {
            ExcelUtil.validateExcelTemplate(headMap, clazz, fields);
        }
    }

    public void offValidate() {
        this.validateSwitch = false;
    }

    public int getRows(){
        return rows;
    }
}
```



### 实体类

```java
@Data
public class MajorHazardStatisticsExcel implements Serializable {
    @ExcelIgnore
    private static final long serialVersionUID = 1L;

    /**
     * 名称
     */
    @ExcelProperty(value = "名称")
    private String name;

    /**
     * 地址
     */
    @ExcelProperty(value = "地址")
    private String address;

    /**
     * 危化品类型
     */
    @ExcelProperty(value = "危化品类型")
    private String chemicalType;

    /**
     * 规模
     */
    @ExcelProperty(value = "规模")
    private String scale;

    /**
     * 法人
     */
    @ExcelProperty(value = "法人")
    private String legalPerson;

    /**
     * 联系方式
     */
    @ExcelProperty(value = "联系方式")
    private String contact;


}

```



### ~~方案一：~~废案

*该方案由于实体类不一致导致导入比较繁琐，所以使用方案二*

```java
@Slf4j
public class ExcelUtil {
    /*
     * 校验excel文件的表头，与数据模型类的映射关系是否匹配
     * */
    public static void validateExcelTemplate(Map<Integer, String> headMap, Class<?> clazz, Field[] fields) {
        Collection<String> headNames = headMap.values();

        // 类上是否存在忽略excel字段的注解
        boolean classIgnore = clazz.isAnnotationPresent(ExcelIgnoreUnannotated.class);
        int count = 0;
        for (Field field : fields) {
            // 如果字段上存在忽略注解，则跳过当前字段
            if (field.isAnnotationPresent(ExcelIgnore.class)) {
                continue;
            }

            ExcelProperty excelProperty = field.getAnnotation(ExcelProperty.class);
            if (null == excelProperty) {
                // 如果类上也存在忽略注解，则跳过所有未使用ExcelProperty注解的字段
                if (classIgnore) {
                    continue;
                }
                // 如果检测到既未忽略、又未映射excel列的字段，则抛出异常提示模板不正确
                throw new ExcelAnalysisException("请检查导入的excel文件是否按模板填写!");
            }

            // 校验数据模型类上绑定的名称，是否与excel列名匹配
            String[] value = excelProperty.value();
            String name = value[0];
            if (name != null && 0 != name.length() && !headNames.contains(name)) {
                throw new ExcelAnalysisException("请检查导入的excel文件是否按模板填写!");
            }
            // 更新有效字段的数量
            count++;
        }
        // 最后校验数据模型类的有效字段数量，与读到的excel列数量是否匹配
        if (headMap.size() != count) {
            throw new ExcelAnalysisException("请检查导入的excel文件是否按模板填写!");
        }
    }
}
```

### 方案二：

##### serviceImpl：

==注意：调用时记得使用==```EasyExcelFactory.read(inputStream,FieldSurveySummaryExcel.class , readListener).sheet().doRead();```

*在使用 ==EasyExcel==库读取Excel文件时，如果不指定 `FieldSurveySummaryExcel.class`（或类似的数据映射类），EasyExcel将无法直接将==Excel文件中的每一行数据映射到 Java对象上==。这个数据映射类通常通过注解来指定Excel文件中每一列与 Java对象字段之间的对应关系。*

```java
 @Override
    @Transactional(rollbackFor = Exception.class)
    public int importExcel(MultipartFile file) {
        int num = 0;
        // 创建通用监听器来解析excel文件
        try (InputStream inputStream = file.getInputStream()) {
            Consumer<List<FieldSurveySummaryExcel>> consumer = this::batchSaveData;
            CommonReadExcelListener<FieldSurveySummaryExcel> readListener = new CommonReadExcelListener<>(FieldSurveySummaryExcel.class,consumer);
            EasyExcelFactory.read(inputStream,FieldSurveySummaryExcel.class , readListener).sheet().doRead();
            num = readListener.getRows();
        } catch (IOException e) {
            Assert.isFalse(false,"导入错误");
        }
        return num;
        // 对读取到的数据进行批量保存
//        List<FieldSurveySummaryExcel> excelData = listener.getData();
//        batchSaveExcelData(excelData);
    }

    private void batchSaveData(List<FieldSurveySummaryExcel> list) {
        List<FieldSurveySummary> data = BeanUtil.copyToList(list, FieldSurveySummary.class);
        this.saveBatch(data);
    }
```



##### 通用监听器

```java
@Slf4j
public class CommonReadExcelListener<T> extends AnalysisEventListener<T> {

    private final static int BATCH_NUM = 1000;
    private final int batchNum;
    private int rows = 0;
    private boolean validateSwitch = true;
    List<T> data = new ArrayList<>();

//    private final IService<T> service;

    private final Consumer<List<T>> business;

    /*
     * 用于校验excel模板正确性的字段
     * */
    private final Field[] fields;
    private final Class<T> clazz;

    public CommonReadExcelListener(Class clazz,Consumer<List<T>> business){
        this(clazz,business,BATCH_NUM);
    }

    public CommonReadExcelListener(Class clazz,Consumer<List<T>> business,int batchNum){
        this.clazz = clazz;
        this.fields = clazz.getDeclaredFields();
        this.business = business;
        this.batchNum = batchNum;
    }

    /**
     * 每解析一行数据就调用该方法
     * @param t 实体
     * @param analysisContext 分析上下文
     */
    @Override
    public void invoke(T t, AnalysisContext analysisContext) {
        data.add(t);
        if (data.size() >= batchNum){
//            service.saveBatch(data,batchNum);
            this.business.accept(data);
            rows += data.size();
//            data.clear();
            data = new ArrayList<>();
        }
    }

    /**
     * 每解析完一页sheet页就会调用该方法
     * @param analysisContext 分析上下文
     */
    @Override
    public void doAfterAllAnalysed(AnalysisContext analysisContext) {
        log.info("读取当前文件第【"+(analysisContext.readSheetHolder().getSheetNo() + 1)+"】sheet页");
        if (data.size() > 0){
//            service.saveBatch(data,batchNum);
            this.business.accept(data);
            rows += data.size();
        }
        data = new ArrayList<>();
    }

    /*
     * 读取到excel头信息时触发，会将表头数据转为Map集合（用于校验导入的excel文件与模板是否匹配）
     *   注意点1：当前校验逻辑不适用于多行头模板（如果是多行头的文件，请关闭表头验证）；
     *   注意点2：使用当前监听器的导入场景，模型类不允许出现既未忽略、又未使用ExcelProperty注解的字段；
     * */
    @Override
    public void invokeHeadMap(Map<Integer, String> headMap, AnalysisContext context) {
        if (validateSwitch) {
            ExcelUtil.validateExcelTemplate(headMap, clazz, fields);
        }
    }

    public void offValidate() {
        this.validateSwitch = false;
    }

    public int getRows(){
        return rows;
    }
}
```

##### 前端

```vue
<el-form label-width="100px">
      <div style="margin-bottom: 10px">
        <el-button type="primary" @click="clickDownloadButton">数据导入模板</el-button>
      </div>
      <el-upload
        ref="uploader"
        v-loading="loading"
        :action="action"
        :before-upload="beforeDataUpload"
        :disabled="loading"
        :headers="headers"
        :on-error="handleError"
        :on-success="handleSuccess"
        class="upload-demo"
        drag
        element-loading-background="rgba(255, 255, 255, 0.3)"
        element-loading-spinner="el-icon-loading"
        element-loading-text="数据导入中，请稍候..."
      >
        <template v-if="!loading">
          <img class="img" src="./img.png" alt="导入文件" />
          <div class="el-upload__text">
            <div>将要导入的文件拖拽到此处</div>
            <div>仅支持 <span style="color: red">.xls或.xlsx</span> 格式</div>
          </div>
        </template>
      </el-upload>
    </el-form>
```



---

# excel模板导出

##### 后端

```java
private void downloadTemplate(HttpServletResponse response) {
    try {
        String excel = "无人机航拍数据信息模板.xlsx";
        String path = String.format("template/%s", excel);
        String filename = URLEncoder.encode(excel, "UTF-8").replaceAll("\\+", "%20");

        // 设置响应类型
        response.setContentType("application/octet-stream");
        // 设置编码格式
        response.setCharacterEncoding("UTF-8");
        // 设置响应头
        response.setHeader("Access-Control-Expose-Headers", "Content-Disposition");
        response.setHeader("Content-Disposition", "attachment;filename=" + filename);

        ClassPathResource resource = new ClassPathResource(path);

        Assert.isTrue(resource.exists());

        IoUtil.copy(resource.getInputStream(), response.getOutputStream());

    } catch (IOException e) {

        throw new RuntimeException("模板下载失败");
    }
}
```

##### 前端

```js
clickDownloadButton() {
      this.$confirm('确定下载模板?', '温馨提示', {
        confirmButtonText: '确认',
        cancelButtonText: '取消',
        type: 'warning'
      }).then(() => {
        axios
          .post(
            `${this.templateUrl}`,
            {},
            {
              responseType: 'arraybuffer'
            }
          )
          .then((res) => {
            let blob = new Blob([res.data], {type: "application/octet-stream"})
            const url = window.URL.createObjectURL(blob) // 设置路径
            const link = window.document.createElement('a') // 创建a标签
            const contentDisposition = res.headers['content-disposition']
            console.log(res.headers)
            let fileName = contentDisposition.split(`=`)[1]
            fileName = decodeURIComponent(fileName)
            link.href = url
            link.setAttribute('download', fileName || '模板.xlsx') // 设置文件名
            link.style.display = 'none'
            link.click()
            URL.revokeObjectURL(url)
          })
          .catch(() => {
            this.$notify.error('模板下载失败')
          })
      })
    }
```









## 导出下载

##### 前端

```js
exportExcel(url, params, name){
      axios.post('/pub/sendSmsDetail/export', params, { responseType: 'blob' }).then((res) => {
        const blob = new Blob([res.data], {
          type: 'application/vnd.ms-excel;charset=UTF-8'
        })
        const url = window.URL.createObjectURL(blob) // 设置路径
        const link = window.document.createElement('a') // 创建a标签
        link.href = url
        link.download = name // 设置文件名
        link.style.display = 'none'
        link.click()
        URL.revokeObjectURL(url)
      })
    }
```

java

```java
try{
            // 设置响应头
            response.setContentType("application/vnd.ms-excel");
            response.setHeader("Content-Disposition", "attachment;filename=短信发送明细.xlsx");
            response.setCharacterEncoding("UTF-8");

            ServletOutputStream outputStream = response.getOutputStream();
            List<SendSmsExport> sendSmsExports = BeanUtil.copyToList(sendSmsDetails, SendSmsExport.class);
            EasyExcel.write(outputStream, SendSmsExport.class).sheet().doWrite(sendSmsExports);
        }catch (Exception e){
            e.printStackTrace();
        }
```

