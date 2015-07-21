---
layout: post
title: Ember data 笔记
date: '2015-07-20T00:00:00.000Z'
categories: front-end
---

**主要摘抄自 emeber-cli 101**

# DS.Store
存储层是与记录和后端交互的主要窗口，数据的创建、加载和删除都在存储层完成，然后再由存储层把变动应用到后端。

## 方法
**注意， find 方法、 all 方法和 filter 方法返回的都是 Promise 对象**

### `all('model')`
返回存储层里已经加载的记录，实时更新

### `filter('model', {attrs..}, filter)`
filter 方法只会处理存储层中已经加载的记录(和 all 方法一样)，如果想强制请求后端，可以在参数中传入一个对象，让 filter 方法查询这个对象中指定的属性

```javascript
friends = $E.store.filter('friend'， function(friend){
  return friend.get('totalArticles') % 2 == 0
})

friends = $E.store.filter('friend'， {hasArticles: true}， function(friend){
  return friend.get('totalArticles') % 2 == 0
})
```

### `findAll('model')`
请求服务器，加载指定模型中的所有记录

```javascript
friends = $E.store.findAll('friend')
// XHR finished loading: GET "http://localhost:4200/api/v2/friends".
```

### `findQuery('model', hash)`
findQuery 方法的第一个参数是模型名，第二个参数是一个对象，对象中的键值对会作为请求的查询参数附加到 URL 中

```javascript
friends = $E.store.findQuery('friend'， {hasArticles: true， sort_by: 'created_at'})
// XHR finished loading: GET "http://localhost:4200/api/v2/friends?hasArticles=true&sort_by=created_at"
```

### `find('model', id)`
加载单条记录，当储存层没有该记录时会请求后端

### `getById('model', id)`
同步查询单条记录，不存在返回null

### `metadataFor('model')`
如果 API 的响应中包含 meta 键，我们可以使用 metadataFor 方法获取这些元数据。这个方法在实现分页等功能时用得到。

### `createRecord('friend', {attrs..})`
创建指定模型的新记录

## 关联
如果想定义"拥有多个"类型的关联，要使用 hasMany 方法;如果要定义"属于"类型的关联，要使用 belongsTo 方法

在 Ember 中处理关联有两种方式，第一种方式是把关联的记录旁载到存储层，第二种方式是按需加载关联的记录。如果使用第一种方式，响应中要有所关联记录的 ID;Ember Data 会寻找这些 ID 对应的记录，然后自动把这些记录加载到存储层。

如果我们知道关联的所有记录都已经加载到存储层了，或者要旁载的记录数很少，适合使用第一种方式

### 异步加载关联
Ember Data 处理异步关联时，首先加载父记录，然后加载关联的记录，不过仅当我们明确获取存储关联记录的属性时才会加载。例如，如果调用 friend.get('articles') ，Ember Data  先检查是否已经加载了 articles 属性中的记录，如果没有，会发起 GET 请求加载这些记录。如果关联的物品 ID 为 40 和 41，Ember Data 请求的地址是 `/api/articles?ids%5B%5D=40&ids%5B%5D=41`

```javascript
// 设置要异步关联的模型
articles: DS.hasMany('articles', {async: true}),
```

访问 `localhost:4000/friends` ， emebr 只会发起请求(`XHR finished loading: GET "http://localhost:4200/api/v3/friends"`)加载 friends 的记录，然后查看某个朋友的个人资料页面，会发起多个 GET 请求获取这个朋友借走的物品

```
XHR finished loading: GET "http://localhost:4200/api/v3/articles/34"
XHR finished loading: GET "http://localhost:4200/api/v3/articles/35"
XHR finished loading: GET "http://localhost:4200/api/v3/articles/36"
XHR finished loading: GET "http://localhost:4200/api/v3/articles/37"
```

我们可以让 Ember Data 合并所有请求。在适配器中把 coalesceFindRequests 属性的值设为 true

```javascript
import DS from 'ember-data';
export default DS.ActiveModelAdapter.extend({
  namespace: 'api/v3',
  coalesceFindRequests: true
})
```

请求就会合并成一个`XHR finished loading: GET "http://localhost:4200/api/v3/articles?ids%5B%5D=34&ids%5B%5D=35&ids%5B%5D=36&ids%5B%5D=16".`

api服务器处理也很简单，以 rails 为例

```ruby
class Api::V3::ArticlesController < Api::BaseController
  def index
    if params[:ids]
      @articles = Article.find(params[:ids])
    else
      @articles = Article.all
    end

    render json: @articles
end
```

### 链接(link)
Ember Data 还支持一种异步加载关联记录的方式。我们可以在后端的响应中设定一个名为 links 的键，这个键的值是一个对象，包含获取各个关联记录的 URL。需要使用关联的记录时，Ember Data 会请求这个 URL。

```
{
  id: 48,
  first_name: "Zombo",
  last_name: "Pombo",
  email: "zombo@pombo.com",
  twitter: "zombo",
  total_articles: 2,
  links: {
    articles: "/api/v4/articles?friend_id=48"
  }
}
```
