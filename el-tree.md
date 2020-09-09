# el-tree操作实现

* 注意参数是data还是node，node的id和data的[key]是不同的

### Q: 更改或者清除当前节点
#### A: `setCurrentNodeKey(key) 或 setCurrentNode(node)`
```javascript
   // key可以是空
   setCurrentNodeKey(key) {
    // 清空选中节点
    if (key === null || key === undefined) {
      // currentNode.isCurrent控制节点is-current样式
      this.currentNode && (this.currentNode.isCurrent = false);
      this.currentNode = null;
      return;
    }
    // 更改选中节点
    const node = this.getNode(key);
    if (node) {
      // 这里的setCurrentNode不接受空参数
      this.setCurrentNode(node);
    }
  }
  // 这里没有做空判断，内部设置isCurrent，为空时报错
  setCurrentNode(currentNode) {
    const prevCurrentNode = this.currentNode;
    if (prevCurrentNode) {
      prevCurrentNode.isCurrent = false;
    }
    this.currentNode = currentNode;
    this.currentNode.isCurrent = true;
  }
```
### Q: 获取节点Node对象
#### A: `getNode(data)`
```javascript
  // 可以直接传送id，或者一个包含key的对象，从nodesMap中获取
  getNode(data) {
    if (data instanceof Node) return data;
    const key = typeof data !== 'object' ? data : getNodeKey(this.key, data);
    return this.nodesMap[key] || null;
  }
```
### Q: 设置选中状态
#### A: `setChecked(data, checked, deep)`
```javascript
  setChecked(data, checked, deep) {
    const node = this.getNode(data);

    if (node) {
      node.setChecked(!!checked, deep);
    }
  }
```
### Q: 设置节点展开状态
#### A: `setDefaultExpandedKeys(keys)`
```javascript
  // 该操作只会展开keys包含的节点，不会折叠其他节点
  setDefaultExpandedKeys(keys) {
    keys = keys || [];
    this.defaultExpandedKeys = keys;

    keys.forEach((key) => {
      const node = this.getNode(key);
      if (node) node.expand(null, this.autoExpandParent);
    });
  }
```
### Q: 批量设置节点选中状态
#### A: `setCheckedNodes(array, leafOnly = false) 或 setCheckedKeys(keys, leafOnly = false)`
```javascript
  setCheckedNodes(array, leafOnly = false) {
    const key = this.key;
    const checkedKeys = {};
    array.forEach((item) => {
      checkedKeys[(item || {})[key]] = true;
    });

    this._setCheckedKeys(key, leafOnly, checkedKeys);
  }

  setCheckedKeys(keys, leafOnly = false) {
    this.defaultCheckedKeys = keys;
    const key = this.key;
    const checkedKeys = {};
    keys.forEach((key) => {
      checkedKeys[key] = true;
    });

    this._setCheckedKeys(key, leafOnly, checkedKeys);
  }
```
> 两个函数做一些标准处理后都会调用`_setCheckedKeys(key, leafOnly = false, checkedKeys)`函数，内部清空所有节点的选中状态后设置目标节点选中状态。
