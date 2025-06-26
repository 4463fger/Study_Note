[toc]

# `ExcelDataReader`

## 下载

https://github.com/ExcelDataReader/ExcelDataReader

`ExcelDataReader`是基础包

- 使用 `reader`接口来读取数据。

`ExcelDataReader.DataSet`是拓展包

- 使用`AsDataSet()`方法来将Excel中的数据填充到`System.Data.DataSet`中。

## How To Use

### 官方例

```csharp
// 打开Excel文件
using (var stream = File.Open(filePath, FileMode.Open, FileAccess.Read))
{
    // 自动检测文件格式,支持以下格式
    // - 二进制Excel文件 (2.0-2003 格式; 拓展名为.xls)
    // - OpenXml Excel文件 (2007 格式; 拓展名为xlsx, 拓展名为.xlsb)
    using (var reader = ExcelReaderFactory.CreateReader(stream))
    {
        //选择1或2两种写法

        // [1]: 使用reader的方法
        do
        {
            while (reader.Read())
            {
                // reader.GetDouble(0);
            }
        } while (reader.NextResult());

        // [2]: 使用 AsDataSet 扩展方法,
        // 将Excel文件中的数据转化为DataSet对象
        var result = reader.AsDataSet();
        // 每个工作表的结果都在result.Tables 中，即可以通过 result.Tables访问 DataSet 中各个工作表的数据
    }
}
```

**假设我们现在有这么一个Excel表格**

```csharp
	包名称	ContentKey	Chinese	English
    主界面	Text_GameStart	开始游戏	PLAY
    主界面	Text_Setting	设置	SETTING
    主界面	Text_ExitGame	退出游戏	EXITGAME
    设置界面	Text_Save	保存	SAVE
    设置界面	Text_AudioButton	音效设置	AUDIO
    设置界面	Text_ViewButton	界面设置	VIEW
    设置界面	Text_GameVolume	全局音量	Global Volume
    设置界面	Text_BGMVolume	音乐	BGM Volume
    设置界面	Text_EffectVolume	音效	Effect Volume
    设置界面	Text_Resolution	分辨率	Resolution
    设置界面	Text_FPS	帧率	FPS
    设置界面	Text_Language	语言	Language
```

```csharp
public class ExcelDataReaderStudy : EditorWindow
{
    private string excelPath ="";

    [MenuItem("Study/ExcelDataReader_Study")]
    private static void ShowWindow()
    {
        var window = GetWindow<ExcelDataReaderStudy>();
        window.titleContent = new GUIContent("ExcelDataReaderStudy");
        window.Show();
    }

    private void OnGUI()
    {
        excelPath = EditorGUILayout.TextField("ExcelPath",excelPath);

        // 选择Excel文件路径
        if (GUILayout.Button("选择文件路径"))
        {
            var path = EditorUtility.OpenFilePanel("Select Excel File", "OK", "xlsx");
            if (!string.IsNullOrEmpty(path))
            {
                excelPath = path;
            }
        }
        
        // [1] 方法一
        // [2] 方法二
    }
}
```

### [1] reader方法读取

> reader.Read()是逐行读取的，然后在每一行中去对列进行处理

```csharp
// [1] : Reader方法读取
if (GUILayout.Button("测试Reader读取器"))
{
    try
    {
        using (FileStream stream = File.Open(excelPath,FileMode.Open,FileAccess.Read))
        {
            // [2] . 创建 ExcelReader(自动识别 .xls 或 .xlsx 格式)
            using (IExcelDataReader reader = ExcelReaderFactory.CreateReader(stream))
            {
                // [3] . 创建列名映射字典
                var columnMap = new Dictionary<string, int>();
                // [4] . 读取第一个工作表
                bool isFirstRow = true;
                do
                {
                    Debug.Log($"正在读取工作表: {reader.Name}");

                    // 逐行读取
                    while (reader.Read())
                    {
                        // [5] . 如果是第一行（表头），创建列名映射
                        if (isFirstRow)
                        {
                            for (int i = 0; i < reader.FieldCount; i++)
                            {
                                string columnName = reader.GetString(i);
                                columnMap[columnName] = i;
                                Debug.Log($"映射列: {columnName} -> 索引 {i}");
                            }
                            isFirstRow = false;
                            continue; // 跳过表头行
                        }

                        // [7] : 使用列明映射获取值
                        string GetValue(string columnName)
                        {
                            if (columnMap.TryGetValue(columnName,out int index) && index < reader.FieldCount)
                            {
                                return reader.GetValue(index)?.ToString() ?? "";
                            }
                            return "";
                        }

                        string bagName = GetValue("包名称");
                        string contentKey = GetValue("ContentKey");
                        string chinese = GetValue("Chinese");
                        string english = GetValue("English");

                        if (!string.IsNullOrEmpty(contentKey))
                        {
                            Debug.Log($"[{bagName}] {contentKey}: {chinese} | {english}");
                        }
                    }
                    isFirstRow = true; // 重置为下一个工作表准备
                } while (reader.NextResult()); // 读取下一个工作表(如果有多个Sheet的话)
            }
        }
    }
    catch (Exception ex)
    {
        Debug.LogError($"读取 Excel 失败: {ex.Message}");
    }
}
```

### [2] AsDataSet方法读取

- `row.Field<string>("列名")`
  - 用于通过 **列名** 直接获取单元格的值
  - 必须设置`ExcelDataTableConfiguration`,使用第一行作为列头

```csharp
// [2] AsDataSet方法读取
if (GUILayout.Button("测试AsDataSet读取器"))
{
    try
    {
        using (FileStream stream = File.Open(excelPath,FileMode.Open,FileAccess.Read))
        {
            using (IExcelDataReader reader = ExcelReaderFactory.CreateReader(stream))
            {
                var dataSet = reader.AsDataSet(new ExcelDataSetConfiguration()
                                               {
                                                   ConfigureDataTable = _ => new ExcelDataTableConfiguration()
                                                   {
                                                       UseHeaderRow = true, // 使用第一行作为列头
                                                   }
                                               });
                // AsDataSet读取整个Excel第一张表
                DataTable dataTable = dataSet.Tables[0];
                Debug.Log($"正在读取工作表: {dataTable.TableName}");

                foreach (DataRow row in dataTable.Rows)
                {
                    // 通过列名获取值
                    string bagName = row.Field<string>("包名称") ?? "";
                    string contentKey = row.Field<string>("ContentKey") ?? "";
                    string chinese = row.Field<string>("Chinese") ?? "";
                    string english = row.Field<string>("English") ?? "";

                    if (!string.IsNullOrEmpty(contentKey))
                    {
                        Debug.Log($"[{bagName}] {contentKey}: {chinese} | {english}");
                    }
                }
            }
        }
    }
    catch (Exception ex)
    {
        Debug.LogError($"读取 Excel 失败: {ex.Message}");
    }
}
```

