# 基于antd实现服务端分页Table穿梭框组件
> 最近接触到的需求有穿梭框，而且大量数据需要分页加载请求，看了一下Antd文档，没有现成的服务端分页参数支持
> 需要自定义穿梭框内部的node渲染

> 参考[Antd-Transfer文档](https://ant.design/components/transfer/#API)
> Transfer accept children to customize render list, using follow props:
```jsx
    <Transfer>
        {{
            direction,  // 列表呈现方向 left | right
            disabled, // 列表是否禁用 boolean
            selectedKeys, // 所选项 RecordType[]
            filteredItems, // 过滤项 string[]
            onItemSelect, // 选择事件触发器 (key: string, selected: boolean)	
            onItemSelectAll // 批量选择事件触发器 (keys: string[], selected: boolean)
        }} => {
            return node
        }
    </Transfer>
```

## 具体需求
- 数据加载使用服务端分页
- 其他操作场景需要和原始的Tansfer操作场景保持一致

## 实现注意项
- `Transfer`中左右侧的数据必须包含在初始的`dataSource`属性中
- 使用自定义`render`数据列表，需要区分`direction`分别渲染`left`和`right`侧的数据，这里就要清楚哪些是`targetKeys`
- 实现的关键要正确的显示：所有可选数据、已选数据、未选数据
- 因为是服务端分页，前端没办法一次性拿到所有数据，这个时候如果初始的targetKeys未在第一页，就需要每次在初始化的时候合并当前已选的数据到整个可选数据列表内，这里要求后端返回的已选数据格式至少为：`[{key,label}]`
- 分页加载，就可以使用`Antd-Table`，注意左右两侧的显示不同，即左侧是分页加载数据服务端分页，而右侧是显示当前所有已经选择的数据列表，是客户端分页
- 自定义渲染`Transfer`内部节点，那么搜索功能也需要自己自定义，可以参照原来的参数方式做
- 搜索功能，也需要区分left和right，left是需要走服务端加载，且查找的结果需要合并right已选list，right直接前端关键词查找
- 考虑使用者的使用习惯，可以参考Transfer组件原来的属性传参，可以暴露targetKeys、onChange、showSearch、保持原来传递的属性有效性
- 简化使用，直接在组件内部做接口加载的工作，使用时只需要传入request方法

## 总结
1. 整体实现不难，主要理解了Transfer组件本身的一些重要属性的意义和使用场景即可
2. 主要的工作量在于，如何让这种分页加载模式的穿梭框，如何在不影响基本功能使用的情况下，尽量还原原来的使用场景的操作方式
3. 这里唯一有问题的地方就是全选、全选当页，这块还没有很好的实现和还原
4. 为了保证组件统一规范，需要基于Transfer组件做改造实现当前的功能，其实还可以直接用原生的来写，这样灵活度更高，而且不需要考虑组件本身的要求和限制，这样在其他更多功能拓展上做得更好。但是这也是一个比较耗时的。

## 代码
```jsx
const TableTransfer = (props) => {
  const { 
    rowKey = 'key', // 唯一键
    filterKey = 'title', // 右侧过滤键
    renderColumns = [{ dataIndex: 'title', ellipsis: true }], // 表格显示列
    fetchRequest, // 查询请求
    showSearch = true, // 是否显示搜索
    targetKeys = [], // 当前已选
    onChange = () => {}, // 当前已选更新
    ...restProp 
  } = props
  const isInit = useRef(1)
  const [rightSearch, setRightSearch] = useState()
  const [state, setState] = useState({
    dataSource: [],
    totalDataSource: [],
    pagination: {
      current: 1,
      pageSize: 10,
      total: 0,
    },
    search: null,
  })

  const setTotalData = (list, ext, isSearch) => {
    // 如果是搜索 默认初始只保留target
    const all = isSearch ? state.totalDataSource.filter((item) => targetKeys.includes(item[rowKey])) : [...state.totalDataSource]
    // 当前查询的data
    for (let i = 0; i < list.length; i++) {
      if (!all.find((item) => item[rowKey] === list[i][rowKey])) {
        all.push(list[i])
      }
    }
    // 之前已选的target
    for (let i = 0; i < ext.length; i++) {
      if (!all.find((item) => item[rowKey] === ext[i][rowKey])) {
        all.push(ext[i])
      }
    }
    return all
  }

  const fetchData = async (params) => {
    if (typeof fetchRequest !== 'function') {
      console.error('fetchRequest should be function')
      return
    }
    const current = params?.current || state.pagination.current || 1
    const pageSize = params?.pageSize || state.pagination.pageSize || 10
    const search = params?.search !== undefined ? params.search : state.search

    const result = await fetchRequest({ pageSize, current, search})
    if(result) {
      const { data, targetData, total, } = result
      if (isInit.current) {
        onChange(targetData.map(item => item[rowKey]))
        isInit.current = 0
      }
      setState({
        ...state,
        search: search,
        dataSource: data.map((item) => ({ id: item.id, title: item.title })),
        totalDataSource: setTotalData(data, targetData, typeof search !== undefined),
        pagination: {
          current: current,
          pageSize: pageSize,
          total: total,
        },
      })
    }
  }

  useEffect(() => {
    fetchData({ pageSize: 10, current: 1 })
  }, [])

  return (
    <Transfer
      {...restProp}
      className={'lj-table-transfer'}
      dataSource={state.totalDataSource}
      rowKey={(record) => record[rowKey]}
      showSearch={false}
      targetKeys={targetKeys}
      onChange={onChange}
    >
      {({ 
        direction, 
        onItemSelect, 
        selectedKeys: listSelectedKeys, 
        disabled: listDisabled 
      }) => {
        const rowSelection = {
          getCheckboxProps: (item) => ({
            disabled: listDisabled || item.disabled,
          }),
          onSelect(obj, selected) {
            onItemSelect(obj[rowKey], selected)
          },
          selectedRowKeys: listSelectedKeys,
        }

        let rightDataSource = state.totalDataSource.filter((item) => {
          let hasKey = targetKeys.includes(item[rowKey])
          if(rightSearch) {
            hasKey = item[filterKey].indexOf(rightSearch) !== -1
          }
          return hasKey
        })

        let leftDataSource = state.dataSource.map((item) => ({
          ...item,
          disabled: targetKeys.includes(item[rowKey]),
        }))

        return (
          <>
            {
              showSearch && 
              <div className="search-wrapper">
                <Input 
                  placeholder="请输入搜索内容" 
                  allowClear 
                  prefix={<SearchOutlined />}
                  onPressEnter={(e) => direction === 'left' ? 
                    fetchData({ current: 1, pageSize: 10, search: e.target.value }) : 
                    setRightSearch(e.target.value.trim()) 
                  } 
                />
              </div>
            }
            <Table
              columns={renderColumns}
              bordered={false}
              rowSelection={rowSelection}
              showHeader={false}
              dataSource={direction === 'left' ? leftDataSource : rightDataSource}
              size="small"
              rowKey={rowKey}
              onRow={({ disabled: itemDisabled, ...item }) => ({
                onClick: () => {
                  if (itemDisabled) return
                  onItemSelect(item[rowKey], !listSelectedKeys.includes(item[rowKey]))
                },
              })}
              pagination={
                direction === 'left'
                  ? {
                      total: state.pagination.total,
                      pageSize: state.pagination.pageSize,
                      current: state.pagination.current,
                      simple: true,
                    }
                  : { simple: true }
              }
              onChange={(page) => direction === 'left' && fetchData(page)}
            />
          </>
        )
      }}
    </Transfer>
  )
}
```