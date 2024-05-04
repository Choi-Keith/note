# awk

## 基本用法
```shell
# 格式
awk 动作 文件名

awk '{print $0}' demo.txt

echo 'this is a test' |awk '{print $0}' # this is a test

```
print $0就是把标准输入this is a test,     
awk会根据空格和制表符，将每一行分成若干字段，依次用$1、$2、$3代表第一个字段、第二个字段、第三个字段等等


## 计算

```shell
awk 'BEGIN{print 1/3}' #0.33333

num1=666
num2=999

awk -vn1=$num1 -vn2=$num2 'BEGIN{print n1*n2}'
awk -vn1=$num1 -vn2=$num2 'BEGIN{print n1/n2}'
awk -vn1=$num1 -vn2=$num2 'BEGIN{print n1+n2}'
awk -vn1=$num1 -vn2=$num2 'BEGIN{print n1-n2}'
```