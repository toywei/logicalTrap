# logicalTrap

logicalTrap 逻辑陷阱

总结代码逻辑问题的案例。

### 不遵守“单一职责”原则

案例

报表数据某几天汇总值为0；

<pre>
def pay_service_day(service):

    service_id = SERVICE.get(service)
    if service == 'direct':
        return pay_direct_day()
    if service == 'search':
        return pay_search_day()
    timestamp_init = mongo_conn_order.find_one({'service_id': service_id}).get('create_timestamp')
    first_day = time.strftime('%Y%m%d', time.localtime(timestamp_init))
    timestamp_init = int(time.mktime(time.strptime(first_day, '%Y%m%d')))
    timestamp_final, timestamp_exists, data_dict = get_finalData('V3_Pay_%s'%service, timestamp_init)
    if timestamp_exists == timestamp_final - 24*3600:
        return data_dict

    for t in range(timestamp_exists+24*3600, timestamp_final, 24*3600):
        datas = mongo_conn_order.aggregate([
            {'$match': {'service_id': service_id, 'create_timestamp': {'$gte':t, '$lt':t+24*3600}, 
                'type_id': {'$in': [1,8,12]}}},
            {'$group': {'_id': '$uid', 'money': {'$sum': '$val'}, 'pay_count': {'$sum': 1}}},
            {'$sort': {'money': 1}}
        ])

        date = time.strftime("%Y%m%d", time.localtime(t))
        datas = list(datas)
        money_total = sum([abs(data['money']) for data in datas])
        money_total = round(abs(money_total), 2)
        data_dict[date] = money_total
        # print(datas)
        detail_dict = {}
        for data in datas:
            if data.get('money') != 0:
                uid = data.get('_id')
                detail_dict[str(uid)] = {
                    'uid': uid,
                    'money_day': abs(data.get('money')),
                    'pay_count': data.get('pay_count'),
                    'pub': data.get('pay_count')
                }

        data_insert_mongo = {
            'name': 'V3_Pay_%s'%service,
            'data': money_total,
            'detail': detail_dict,
            'date': t
        }
        mongo_conn.insert(data_insert_mongo)
    return data_dict
</pre>

1、web端第一次访问原始数据时，在同一个方法中，对原始数据进行加工计算得到报表数据，报表数据入库；

2、之后web端访问，不再更新报表数据，且读取显示第一次的计算结果；

异常原因：

当第一次访问出现在某时间段原始数据残缺或不存在的情况下时，其计算结果为0，写入汇总表，而当该时间段内的数据完整时，无法更新第一次的错误计算结果；

### 将数据库最晚入库数据的时间作为业务数据的最晚时间 

逻辑绑架

【错】“查找最后一条数据的记录时间”【错】

【错】将id最大数据的“数据时间”（不是入库时间）做为表table的行row、集合collection的document文档所代表的业务的时间的最值【错】

【正】业务时间的最值，只能通过业务自身的时间属性字段获取，而非依赖数据库字段【正】

