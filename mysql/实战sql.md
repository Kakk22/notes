批量更新 case when

```sql
UPDATE course
    SET name = CASE id 
        WHEN 1 THEN 'name1'
        WHEN 2 THEN 'name2'
        WHEN 3 THEN 'name3'
    END, 
    title = CASE id 
        WHEN 1 THEN 'New Title 1'
        WHEN 2 THEN 'New Title 2'
        WHEN 3 THEN 'New Title 3'
    END
WHERE id IN (1,2,3)
```

批量插入mybatis 写法

```xml
<update id="delivery">
    update oms_order
    set delivery_sn = case id
    <foreach collection="list" item="item">
        when #{item.orderId} then #{item.deliverySn}
    </foreach>
    end,
    delivery_company = case id
    <foreach collection="list" item="item">
        when #{item.orderId} then #{item.deliveryCompany}
    </foreach>
    end,
    delivery_time = case id
    <foreach collection="list" item="item">
        when #{item.orderId} then now()
    </foreach>
    end,
    status = case id
    <foreach collection="list" item="item">
        when #{item.orderId} then 2
    </foreach>
    end
    where
    id in
    <foreach collection="list" item="item" separator="," open="(" close=")">
        #{item.orderId}
    </foreach>
    and status = 1
</update>
```