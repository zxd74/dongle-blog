# 背景
最近使用Ptyon爬虫数据时，遇到对象无法转换为JSON序列化对象问题`TypeError: Object of type StockData is not JSON serializable`

意思就是对象不能转换为JSON序列化对象，原因是对象没有自定义实现转换为JSON序列化对象的方法没有。

# 实现
```python
class Main(object):
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def __str__(self):
        return 'Main(name={}, age={})'.format(self.name, self.age)

    def __repr__(self):
        return 'Main(name={}, age={})'.format(self.name, self.age)

    def to_json(self):
        return {'name': self.name, 'age': self.age}

    @classmethod
    def from_json(cls, json_data):
        return cls(**json_data)


if __name__ == '__main__':
    main = Main('zhangsan', 18)

    # 因为Main是自定义对象，如果没有指定default方法，则无法转换为JSON序列化对象，并且报错`TypeError: Object of type StockData is not JSON serializable`
    # jsonStr = json.dumps(main)

    jsonStr = json.dumps(main, default=lambda obj: obj.__dict__)
    print(jsonStr)

    json.loads(jsonStr, object_hook=Main.from_json)

```
# 总结
1. 自定义对象转Json时需要自定义转换方法`default`，通常是将对象转为`dict`类型
   1. 可自定义`dict数据内容`
   2. 也可通过`object.__dict__`获取对象属性
2. Json数据转换对象时需要自定义转换方法`object_hook`
   1. `object_pairs_hook`是有序数据方法，优先于`object_hook`
   2. `object_hook`是无序数据方法
```python
# 将json数据转换为对象
def handler(data):
    return Main(data['name'],data['age'])

data = json.loads(jsonStr,object_hook=handler)


# 将对象数据转换为json字符串
def obj_2_json(obj):
    """
    # 自定义类转json需要自定义一个转换成python基本类型的方法
    def obj_2_json(obj):
        return {
            "name":obj.name
        }
    """
    return json.dumps(obj,default=lambda obj:obj.__dict__)
```