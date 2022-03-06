-- 去重  


	delete from t_secondary_mat where id in (select min(id) from (
	    select id,material_code from t_secondary_mat t where material_code in (
	        select material_code from t_secondary_mat group by  material_code having  count(1) >=2
	                                                                  )) as temp group by temp.material_code);

--设置字段值自增

	set @rownum=0;
	update t_primary_order_dw
	SET MANDT = (select  @rownum := @rownum +1)
	       where  MANDT=800


-- 关联表更新

	UPDATE t_primary_order_dw2 d
	INNER JOIN (SELECT  PRODUCT_ID,max(BILLING_DOC_NO) as BILLING_DOC_NO
	FROM t_primary_order_dw2  
	GROUP BY PRODUCT_ID) t
	set d.delete_stauts=0
	WHERE  d.PRODUCT_ID=t.PRODUCT_ID and d.BILLING_DOC_NO=t.BILLING_DOC_NO;
