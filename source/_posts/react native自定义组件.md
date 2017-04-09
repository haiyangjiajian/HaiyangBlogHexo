---
layout: post
title: react native自定义组件
tags: [react_native]
category: 编程
---

react native在app开发上的一个优势就是组件化开发，当有了足够多的自定义组件后，可以很方便的将这些组件拼装起来，开发效率提高很多。本文中将以一个picker为例子，来讲如何包装一个通用性很强的react native组件。本文的示例源码可以在[ReactNativeUIComponents](https://github.com/haiyangjiajian/ReactNativeUIComponents)下载。这个项目未来会不断的维护加入更多实用的组件，也力求这个项目可以作为一个react native的示例项目，包含了react native开发中常用的router，navigator，code push等，可以作为参考。

---

### picker的效果


<img src="{{site.url}}/public/img/rn/picker1.png" width = "300" height = "200" alt="native view"/>

<img src="{{site.url}}/public/img/rn/picker2.png" width = "300" height = "200" alt="native view"/>


其中picker要展示的内容，百度、搜狗、谷歌等均可以由父组件指定，其样式也可以由父组件指定。

### 组件包装

#### 界面

本文封装了一个叫Picker的组件。一般推荐通过定义defaultProps来告诉组件的调用者，可以传入这个组件中的参数有哪些。可以看到可以传入的参数有style：组件样式；animationType：动画类型；transparent：是否透明；modalVisible：是否可见；dataArray：显示的数据；title：标题。可以通过传入这些参数，使这个组件拥有不同的样式，显示不同的数据。


``` javascript
static defaultProps = {
    style: View.propTypes.style,
    animationType: 'none',
    transparent: true,
    modalVisible: true,
    dataArray: [],
    title: 'title'
  };

  constructor(props) {
    super(props);

    this.state = {
      dataSource: this._getDataSource(props.dataArray),
      style: this.props.style,
      animationType: this.props.animationType,
      transparent: this.props.transparent,
      modalVisible: this.props.modalVisible,
    };
  }
```

组件内部的渲染代码如下，主体是一个Modal，然后是一个整体的view

``` javascript
<View style={[styles.modalContainer, {backgroundColor: 'rgba(0, 0, 0, 0.5)'}]}>
```
在这个view中主要有两部分内容，一个是顶部的title，另一个是底部的listview，title显示从父组件中传入的标题，listview显示传入的dataArray。

``` html
<Modal
        animationType={this.state.animationType}
        transparent={this.state.transparent}
        visible={this.state.modalVisible}
        onRequestClose={() => {this._setModalVisible(false)}}>
        <View style={[styles.modalContainer, {backgroundColor: 'rgba(0, 0, 0, 0.5)'}]}>
          <View style={styles.viewContainer}>
            <View style={styles.titleRow}>
              <TouchableOpacity style={styles.cancelButton} onPress={() => {this._setModalVisible(false)}}>
                <Image style={styles.image}
                       source={require('../img/icon_cancel_grey.png')}/>
              </TouchableOpacity>
              <Text style={[styles.titleText, styles.modalTitle]}>{this.props.title}</Text>
            </View>
            <View style={[{marginTop: 0, marginBottom: 0}]}/>
            <ListView
              style={styles.flex}
              dataSource={this.state.dataSource}
              renderRow={(rowData, sectionID, rowID) => this._renderRow(rowData, sectionID, rowID)}
              renderScrollComponent={props => <RecyclerViewBackedScrollView {...props} />}
              renderSeparator={(sectionID, rowID) => <View key={`${sectionID}-${rowID}`} style={[GlobalStyles.divider, {marginTop: 0, marginBottom: 0, marginLeft: 16}]}/>}
            />
          </View>
        </View>
</Modal>
```

listView中每一个cell的渲染

``` javascript
 _renderRow(rowData, sectionID, rowID) {
    return (
      <TouchableHighlight onPress={() => this._pressRow(rowData, rowID)} underlayColor='gray'>
        <View>
          <View style={styles.row}>
            <Text style={styles.rowTitle}>
              {rowData.name}
            </Text>
          </View>
        </View>
      </TouchableHighlight>
    );
  }
```

下面是对这个组件调用的一个示例，puperseItems是一个格式化对象的数组，这个对象包含了name,id,url三个属性


``` html
<Picker
            title="选择搜索引擎"
            dataArray={purposeItems}
            selectedData={(purpose) => {this.onPurposeSelected(purpose);}}
            onHideModal={() => this.setState({purposeModalVisible: false})}
/>
```

至此界面的展示完毕

#### 操作

 这样一个组件主要有两种操作，点击当前列和关闭modal，如下两个函数，主要是调用了父组件传递进来的selectedData和onHideModal回调

``` javascript
  _pressRow(rowData, rowID) {
    this.props.selectedData && this.props.selectedData(rowData);
    this.props.selectedIndex && this.props.selectedIndex(rowID);
  }

  _setModalVisible(visible) {
    this.setState({modalVisible: visible});
    if (visible) {
      this.props.onShowModal && this.props.onShowModal();
    } else {
      this.props.onHideModal && this.props.onHideModal();
    }
  }
```

至此完成了一个组件的封装，可以灵活的显示父组件传入的数据，并进行各种交互。

本文的示例源码可以在[ReactNativeUIComponents](https://github.com/haiyangjiajian/ReactNativeUIComponents)下载










