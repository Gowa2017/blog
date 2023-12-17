---
title: Java中final语义
categories:
  - Java
date: 2019-01-09 14:07:09
updated: 2019-01-09 14:07:09
tags: 
  - Java
---
使用场景是我在安卓组件的点击事件回调中不准备修改 Activity 中的一个 Map 的内容，但是开始的时候，提示我在内部类中访问外部变量必须声明为 final。
<!--more-->

# 重现问题

```java
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_com_register_multi_cert_global_freight_srv);
        ButterKnife.bind(this);
        addCheckBoxToGl(cargoClassArr, cargoClassMap, glCargoClass);

    }
    private void addCheckBoxToGl(String[] data, final Map<String, String> dataMap, GridLayout gl) {
        for (int i = 0; i < data.length; i++) {
            String s = data[i];
            AppCompatCheckBox cb = new AppCompatCheckBox(this);
            cb.setText(s);
            cb.setOnCheckedChangeListener(new CompoundButton.OnCheckedChangeListener() {
                @Override
                public void onCheckedChanged(CompoundButton buttonView, boolean isChecked) {
                    if (isChecked) {
                        dataMap.put(buttonView.getText().toString(), buttonView.getText().toString());
                    } else {
                        dataMap.remove(buttonView.getText().toString());
                    }
                    Log.d(TAG, "onViewClicked: " + new Gson().toJson(cargoClassMap));
                }
            });
            gl.addView(cb);
        }

    }
```

错误提示：**变量 dataMap 从内部类中进行访问，需要声明为 final**

我当时以为，final 是不可改变的意思。但事实证明这是错误的。在进行 final 声明后，依然改变了 dataMap 的值。

最后查阅官方文档才发现：[https://docs.oracle.com/javase/tutorial/java/javaOO/classvars.html](https://docs.oracle.com/javase/tutorial/java/javaOO/classvars.html)

> The final modifier indicates that the value of this field cannot change.
> final 修饰符只是表明，字段的值不能被改变


对于一个引用字段，其值并不是一个对象，而是一个引用。所以改变对象，其引用依然不变，所以可以改变对象的值。

