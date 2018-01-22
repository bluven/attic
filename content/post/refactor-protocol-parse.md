---
title: "记一次项目中协议解析代码重构"
date: 2018-01-20T21:40:34+08:00
---

现在项目里的协议解析很混乱，而且不能复用父类的解析代码，新人扩展协议很麻烦，尤其会把代码写的很丑陋，比如下面这样：

要解析的协议格式：

    <!-- lang: xml -->
    <SyncUploadRequestUnit>
        <Parameter name="ContentName" value="L1NhbXBsZS9iYWNrdXAudGFyLmd6LjA="/>
        <Parameter name="DataType" value="U2FtcGxl"/>
        <Parameter name="PublishDate" value="2013-12-04"/>
        <Parameter name="EffectiveDate" value="2013-12-04"/>
        <Parameter name="EffectiveTime" value="07:47:00"/>
        <Parameter name="FileName" value="L1NhbXBsZS9iYWNrdXAudGFyLmd6"/>
        <Parameter name="UnassignFlag" value="N"/>
        <Parameter name="CRCValue" value="766D29E5"/>
        <Parameter name="AssignmentId" value=""/>
        <Parameter name="Title" value="YmFja3Vw"/>
        <Parameter name="Keywords" value=""/>
        <Parameter name="Version" value=""/>
        <EfbList>
          <Parameter name="EfbName" value="EFB-1"/>
          <Parameter name="EfbName" value="EFB-2"/>
        </EfbList>
        <DepList>
          <Content>
            <Parameter name="DataType" value="U2FtcGxl "/>
            <Parameter name="FileName" value="dGVzdDIudHh0 "/>
            <Parameter name="Revision" value="MjIy "/>
          </Content>
          <Content>
            <Parameter name="DataType" value="L1NhbXBsZS9iYWNrdXAudGFyLmd6LjA="/>
            <Parameter name="FileName" value="dGVzdDIudHh0 "/>
            <Parameter name="Revision" value="MjIy "/>
          </Content>
        </DepList>
    </SyncUploadRequestUnit>

解析代码，每个子类都得实现自己的：
```python

def parse(data, encoded=True):

    syncUnitDOM = parseXmlString(data)
    syncUnitNode = syncUnitDOM.childNodes[0]
    
    paramMap = {}
    efbList = []
    depList = []

    for syncUnitChildNode in syncUnitNode.childNodes:

        if syncUnitChildNode.nodeName == PARAMETER_TAG:
            name = str(syncUnitChildNode.getAttribute(NAME_TAG))
            value = str(syncUnitChildNode.getAttribute(VALUE_TAG))
            paramMap[name] = value
            
        elif syncUnitChildNode.nodeName == EFBLIST_TAG:

            efbListParameterList = syncUnitChildNode.getElementsByTagName(PARAMETER_TAG)

            for efbListNode in efbListParameterList:
                if str(efbListNode.getAttribute(NAME_TAG)) == EFB_NAME:
                    efbList.append(str(efbListNode.getAttribute(VALUE_TAG)))

        elif syncUnitChildNode.nodeName == DEPLIST_TAG:

            for contentNode in syncUnitChildNode.childNodes:

                if contentNode.nodeName == CONTENT_TAG:

                    content = {}

                    for conParamNode in contentNode.getElementsByTagName(PARAMETER_TAG):

                        paramName = str(conParamNode.getAttribute(NAME_TAG))

                        if paramName in ENCODED_TAGS and encoded:
                            content[paramName] = decode(str(conParamNode.getAttribute(VALUE_TAG)))
                        else:
                            content[paramName] = str(conParamNode.getAttribute(VALUE_TAG))
    
                    depList.append(content)

```

不管别人舒不舒服，我受不了了。这种情况必须得改善，一开始想到的是go 语言解析xml那种方式，后来又觉得像声明orm model那样更方便，看了peewee这个orm框架后就开工了，下面是一个model的描述：

```python
    
class SyncUploadRequestUnit(Model):

    content_name = ParameterField('ContentName')

    data_type = ParameterField('DataType')

    publish_date = ParameterField('PublishDate')

    effective_time = ParameterField('EffectiveTime')

    effective_date = ParameterField('EffectiveDate')

    filename = ParameterField('FileName')

    unassign_flag = ParameterField('UnassignFlag')

    crc_value = ParameterField('CRCValue')

    assignment_id = ParameterField('AssignmentId')

    title = ParameterField('Title')

    keywords = ParameterField('Keywords')

    version = ParameterField('Version')

    efb_list = StringListField('EfbList', 'EfbName')

    dep_list = ModelListField('DepList', Content)
```


看着很直观是不是，下面是其基类

```python
    
import base64
import xml.etree.ElementTree as ET


BEGIN_TAG = 'CM'
PARAMETER_TAG = 'Parameter'
ENCODED_TAGS = ['DataType', 'FileName', 'ContentName', 'Revision', 'Title']

def encode(value):
    return base64.encodestring(value.encode('utf-8'))

def decode(value):
    return base64.decodestring(value).decode('utf-8')

class FieldDescriptor(object):

    def __init__(self, name, field):

        self.name = name
        self.field = field

    def __get__(self, instance, cls):

        value = instance._data.get(self.field.tag, None)

        return value 

    def __set__(self, instance, value):

        instance._data[self.field.tag] = value


class TagField(object):

    def __init__(self, tag):
        self.tag = tag\

    def parse(self, root, setter, encoded=False):

        for element in root.iterfind(self.tag):

            setter(self.tag, element.text, encoded)

    def transfer_to_xml(self, parent, getter, encoded=False):

        element = ET.SubElement(parent, self.tag)

        element.text = getter(self.tag, encoded) or ''


class ParameterField(object):

    def __init__(self, name):
        self.tag = name

    def parse(self, root, setter, encoded=False):

        for element in root.iterfind(PARAMETER_TAG):

            setter(element.get('name'), element.get('value'), encoded)

    def transfer_to_xml(self, parent, getter, encoded=False):
        attrs = {
                 'name': self.tag, 
                 'value': getter(self.tag, encoded)
                }

        ET.SubElement(parent, PARAMETER_TAG, attrs)


class StringListField(object):

    def __init__(self, container_name, element_name):

        self.tag = container_name
        self.element_name = element_name

    def parse(self, root, setter, encoded=False):

        children = []

        setter(self.tag, children)

        container = root.find(self.tag)

        if container is None:
            return

        for element in container.iterfind(PARAMETER_TAG):

            children.append(element.get('value'))

    def transfer_to_xml(self, parent, getter, encoded=False):

        value_list = getter(self.tag, encoded)

        if value_list is None or len(value_list) == 0:
            return

        parent = ET.SubElement(parent, self.tag)

        for value in value_list:
            attrs = {
                     'name': self.element_name, 
                     'value': value
                    }

            ET.SubElement(parent, PARAMETER_TAG, attrs)


class ModelListField(object):

    def __init__(self, container_name, model):
        self.tag = container_name
        self.model = model
        self.model_tag = model.__name__

    def parse(self, root, setter, encoded=False):

        children = []

        setter(self.tag, children)

        container = root.find(self.tag)

        if container is None:
            return

        for element in container.iterfind(self.model_tag):

            child = self.model()

            child.parse_element(element, encoded)

            children.append(child)


    def transfer_to_xml(self, parent, getter, encoded=False):

        value_list = getter(self.tag, encoded)

        if value_list is None or len(value_list) == 0:
            return

        parent = ET.SubElement(parent, self.tag)

        for value in value_list:

            value.as_xml(parent, encoded)


class ModelField(object):

    def __init__(self, model):
        self.model = model
        self.tag = model.__name__

    def parse(self, root, setter, encoded=False):

        element = root.find(self.tag)

        if element is not None:

            child = self.model()
            child.parse_element(element, encoded)
            setter(self.tag, child)

    def transfer_to_xml(self, parent, getter, encoded=False):

        value = getter(self.tag, encoded)

        if value is not None:

            value.as_xml(parent, encoded)


FIELD_LIST = [TagField, ParameterField, StringListField, ModelField, ModelListField]


class BaseModel(type):

    def __new__(cls, name, bases, attrs):

        if not bases:
            return super(BaseModel, cls).__new__(cls, name, bases, attrs)

        fields = {
                TagField: [],
                ParameterField: [],
                ModelField: [],
                StringListField: [],
                ModelListField: [],
            }

        for attr, field in attrs.items():

            field_class = type(field)

            if field_class in fields:
                
                fields[field_class].append(field)
                attrs[attr] = FieldDescriptor(attr, field)
                
        attrs['_fields'] = fields

        return super(BaseModel, cls).__new__(cls, name, bases, attrs)


class Model(object):

    __metaclass__ = BaseModel

    def __init__(self, data=None):

        self._data = data or {}

    def parse(self, data, encoded=False):

        return self.parsestring(data, encoded)

    def parsestring(self, data, encoded=False):

        root = ET.fromstring(data)

        return self.parse_element(root, encoded)

    def parse_element(self, root, encoded=False):

        for field_list in self._fields.values():

            for field in field_list:

                field.parse(root, self.__set, encoded)

        return self

    def as_xml(self, parent=None, encoded=False):
    
        tag = self.__class__.__name__

        if parent is not None:
            parent = ET.SubElement(parent, tag)
        else:
            parent = ET.Element(BEGIN_TAG)

        for field_class in FIELD_LIST:

            for field in self._fields.get(field_class, []):

                field.transfer_to_xml(parent, self.__get, encoded)

        return parent

    def as_pretty_xml(self, encoded=False):

        tree = self.as_xml(encoded=encoded)

        self.indent(tree)

        return ET.tostring(tree)

    def indent(self, elem, level=0):

        i = "\n" + level * " "

        if len(elem):

            if not elem.text or not elem.text.strip():
                elem.text = i + " "

            if not elem.tail or not elem.tail.strip():
                elem.tail = i

            for elem in elem:
                self.indent(elem, level+1)

            if not elem.tail or not elem.tail.strip():
                elem.tail = i
        else:
            if level and (not elem.tail or not elem.tail.strip()):
                elem.tail = i

    def __set(self, name, value, encoded=False):

        if encoded and name in ENCODED_TAGS:
            value = decode(value)

        self._data[name] = value

    def __get(self, name, encoded=False):

        value = self._data.get(name, None)

        if encoded and value and name in ENCODED_TAGS:
            value = encoded(value)

        return value
```
