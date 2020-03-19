---
title: XML混合内容反序列化（JAXB）
date: 2019-11-25 18:11:05
tags: [序列化, XML]
categories: 后端开发
---

正常来说对XML进行反序列化是比较简单的，而本次需要解析的XML报文是带有具有混合内容的复合类型XML元素，也就是说报文中的某个元素既含有文本也包含其他元素。 类似于如下文本（ageInfo元素）：

```xml
<student> 
  <ageInfo>My age is 
    <age>17</age> 
  </ageInfo>  
  <name>Kan</name> 
</student>
```

选择使用JAXB对XML报文进行反序列化。上诉报文对应的实体类如下：

```java
@XmlRootElement(name = "student")
public class Student {

    private String name;

    private AgeInfo ageInfo;

    public String getName() {
        return name;
    }

    @XmlElement
    public void setName(String name) {
        this.name = name;
    }

    public AgeInfo getAgeInfo() {
        return ageInfo;
    }

    @XmlElement
    public void setAgeInfo(AgeInfo ageInfo) {
        this.ageInfo = ageInfo;
    }

    @Override
    public String toString() {
        return new StringJoiner(", ", Student.class.getSimpleName() + "[", "]")
                .add("name='" + name + "'")
                .add("ageInfo=" + ageInfo)
                .toString();
    }

    public static class AgeInfo {

        private List<String> text;

        private Integer age;

        public List<String> getText() {
            return text;
        }

        @XmlMixed
        public void setText(List<String> text) {
            this.text = text;
        }

        public Integer getAge() {
            return age;
        }

        @XmlElement
        public void setAge(Integer age) {
            this.age = age;
        }

        @Override
        public String toString() {
            return new StringJoiner(", ", AgeInfo.class.getSimpleName() + "[", "]")
                    .add("text=" + text)
                    .add("age=" + age)
                    .toString();
        }
    }
}
```

JAXB对混合内容提供了`@XmlMixed`注解进行处理，反序列化代码如下：

```java
        Student student;
        try {
            JAXBContext jaxbContext = JAXBContext.newInstance(Student.class);
            Unmarshaller jaxbUnmarshaller = jaxbContext.createUnmarshaller();
            student = (Student) jaxbUnmarshaller.unmarshal(new StringReader(XML));
            LOGGER.info("\n{}", student.toString());
        } catch (JAXBException e) {
            LOGGER.warn("解析XML异常：", e);
        }
```

JAXB对混合内容元素进行处理，元素中的文本会转为`List<String>`集合，在本例中，解析后的集合如下：

![](image-20191126105417573.png)

因元素age分为了两个`String`。文本内容包括了换行符和空格，如果需要忽略的话可以在反序列化前使用正则表达式对报文进行处理。