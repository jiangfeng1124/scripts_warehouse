# Sony SRL系统后续改进工作 #
#### Author: Jiang Guo (jguo@ir.hit.edu.cn)
---
## 1. tree.hh中使用到的接口 ##
---
### 文件：Sentence.h, Sentence.cpp ###

类名：SRLTree

功能：建树

接口：
- `iterator`
- `set_head`
- `append_child`

---
#### 文件：FeatureExtractor.h, FeatureExtractor.cpp

类名：FeatureExtractor

功能：特征计算

接口：
- `post_order_iterator` (`begin_post`, `end_post`)
- `parent`
- `is_valid`
- `node_visited_flags`
- `number_of_children`

---

## 2. Interface Optimization ##
---
修改目标：将Model与Processing两个功能模块从SrlEngine中分离。

修改建议：
- 主要修改SrlEngine.h，只需将`m_prg_classifier`以及`m_srl_classifier`提取出来，放在一个新的class中（如SrlModel）。然后在SrlEngine中声明一个SrlModel成员变量即可。

---

## 3. Merge Model Files ##
---
修改目标：将 `PI/PC/AIAC` 三类模型Merge成一个文件，同时，实现相应的load_model接口。

当前模型组织方式：

- `PI` --> `prg_model/prg.model`
- `PC` --> `pcl_models/*` 每一个谓词对应一个单独的模型，谓词列表在`pcl_models/models.list`中
- `AIAC` --> `srl_model/srl.model`

每个模型文件的格式都是统一的，如下：
```
Label Feature_Name Weight
```
模型文件的行数即为特征的维度。
因此，Merge之后的模型文件可以如下方式进行组织：
```
###PRG-Model <line-number>
...
###PCL-Model-坚持 <line-number>
...
###PCL-Model-反映 <line-number>
...
###SRL-Model <line-number>
...
```
其中line-number指的是后面跟的行数，也就是该模型的特征维度。
如果以二进制格式进行模型文件读写的话，line-number可以改为偏移量。

Sample code for merging models:
```python
Input: prg_model_path
       pcl_models_dir
       srl_model_path
       merged_model_path

def scan(model_path):
    ''' to obtain the number of lines(features) in model
    '''
    cnt_lines = 0
    for l in open(model_path):
        cnt_lines += 1
    return cnt_lines

output = open(merged_model_path, "w")
# merge prg(PI) model
cnt_lines = scan(prg_model_path)
print >> output, "%s\t%d" % ("###PRG-MODEL", cnt_lines)
for l in open(prg_model_path):
    print >> output, l.strip()

# merge pcl(PC) models
pcl_model_names = []
for l in open(os.path.join(pcl_models_dir, "models.list")) :
    pcl_model_names.append(l.strip())
for pcl_model in pcl_model_names:
    cnt_lines = scan(pcl_model)
    predicate_name = os.path.basename(pcl_model)
    print >> output, "%s-%s\t%d" % ("###PCL-MODEL", predicate_name, cnt_lines)
    for l in open(pcl_model):
        print >> output, l.strip()

# merge srl(AIAC) models
cnt_lines = scan(srl_model_path)
print >> output, "%s\t%d" % ("###SRL-MODEL", cnt_lines)
for l in open(srl_model_path):
    print >> output, l.strip()
```

那么，在`SrlModel`（参考Interface Optimization）类中，应定义一个新的模型加载（如：load_merged_model）函数。
```cpp
------pseudo code------
void load_merged_model(merged_model_path)
{
    while (getline(merged_model_path, str))
    {
        if (str.startswith("###PRG-MODEL")) {
            // get start -> end line numbers
            m_prg_classifier = new Classifier(merged_model_path, prg_start, prg_end)
        }
        else if (str.startswith("###PCL-MODEL")) {
            // get predicate name 
            // get start -> end line numbers
            m_pcl_classifiers[predicate] = new Classifier(merged_model_path, pcl_start, pcl_end)
        }
        else if (str.startswith("###SRL-MODEL")) {
            // get start-> end line numbers
            m_srl_classifier = new Classifier(merged_model_path, srl_start, srl_end)
        }
    }
}
```

由于每个模型都是一个`Classifier`（`Classifier.h`），而根据上面的伪码可知，我们修改了`Classifier`的构造函数以及load函数：增加了两个参数。原来在加载模型时，我们是将整个文件里的内容视作一个模型，而现在，我们需要先确定某个模型的起始以及结束位置，将之间的内容加载到对应的模型中。

因此，我们需要修改`Classifier.h`

```cpp
原：Classifier(const std::string& model)
新：Classifier(const std::string& model, size_t start, size_t end) // add two arguments

原：void load_model(const std::string& model_path)
新：void load_model(const std::string& model_path, size_t start, size_t end) // add two arguments
```
由于`load_model`里调用的是最大熵模块的模型加载函数`load`，因此，我们还需要修改一下`maxent.h`以及`maxent.cpp`，为了不影响原有的最大熵模块接口，我建议新增一个load函数，如`load_slice`，供`Classifier`类调用。
```cpp
## maxent.h
class ME_Model
{
    ...
    bool load_slice(const std::string& modelfile, size_t start, size_t end);
    ...
}

## maxent.cpp
bool ME_Model::load_slice(const std::string& modelfile, size_t start, size_t end)
{
    // almost the same with with ME_Model::load function
    // the only difference:
    //    * skip the lines from 0 -> start, end+1 -> eof
}
```


