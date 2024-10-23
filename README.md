## SQL_Archived Scripts

### Business_Volume_Data_Source
```
declare @StartIRP as varchar(10)
set @StartIRP='2018-01'

declare @TY as varchar(10)
set @TY=year(GETDATE())

SELECT top(50000)
	 m.[Purch Doc No] as PO
	,m.[Style Key] as Style_Code
        ,concat(m.[Style Key],m.[Color (Coex) Key]) as 'Option'
	,m.[Ship Win] as ShipWindow
        ,m.[Act Ship Cond Text]

    -------Product Mapping--------
	,m.[Div Set] as Prod_Label
	,case when m.[Div Set] in ('WCA','EDC','WCO','TRD') then 'Women'
              when m.[Div Set] in ('MCA','MDC','MCO') then 'Men'
	      when m.[Div Set] ='ACC' then 'ACC'
	else 'LIFESTYLE' end as Prod_Gender
	,m.[Supply Category] as Prod_SupplyCategory
	,m.[Mat Grp] as Prod_MaterialGroup
	,m.[Prod Cls Text] as Prod_Cluster_Name
	
    -------IRP formatting--------
	,concat(right(m.[IRP CalYMth],4),'-',left(m.[IRP CalYMth],2)) as IRP
	,format(cast(left(m.[IRP CalYMth],2) as numeric), '00') as [IRPMonth]
	,concat('FY',
        case when right(m.[IRP CalYMth],4)=@TY-1 then @TY-1
        when right(m.[IRP CalYMth],4)=@TY-2 then @TY-2
		    else @TY end ) as [New_FY]  
        ,Concat('FY',[IRP FY Key],'/',[IRP FY Key]+1) as Old_FY
 	-------loc--------
	,m.[Trading Company] as SourcingOffice 
	,m.[Ctry of Orig] as CountryOfOrigin
	,m.[Vendor Key] as Vendor_Code
	,m.[Vendor Text] as Vendor_Name
        ,m.[Vendor Country] as Vendor_Country
	,cast (m.[Purch Doc Entry Date] as date) as [PurchDocEntry_Date]
	,m.[POF Key] as POF_Code
        ,m.[POF Text] as POF_Name
	,m.[Act Ship Cond Key] as Act_ShipCondition_Key
	,m.[Act Ship Cond Text] as Act_ShipCondition_Name
	-------Key figure--------
	,[EES PUR@PCS] as EES_PUR_PCS
	,[EES PUR@FOB] as EES_PUR_FOB
	,[EES PUR@PCSAB] as EES_PUR_PCS_AB
        ,[EES REC@PCS] as EES_REC@PCS
        ,[EES PUR@FOB D/C] as EES_PUR@FOB_DC
	,[EES PUR@FOB/PCS] as [EES_PUR_FOB/PCS]
	,[EES PUR@LLC] as [EES_PUR_LLC]
	,[EES PUR@LLC/PCS] as [EES_PUR_LLC/PCS]
	-------Date--------
        ,[Act Dt ZF (Goods Ready Date)] as 'CRD'
        ,[Plan Dt ZG (Cargo Receipt at Hub)] as 'Plan Dt ZG'
        ,[Con Dt ZG (Cargo Receipt at Hub)] as 'Con Dt ZG'
        ,[Rev Dt ZG (Cargo Receipt at Hub)] as 'Rev Dt ZG'
        ,[Act Dt ZG  (Cargo Receipt at Hub)] as 'Act Dt ZG'
FROM [BSMD_Data].[Rep].[BSMD_AllData] as m
where 
concat(right(m.[IRP CalYMth],4),'-',left(m.[IRP CalYMth],2)) >= @StartIRP and [Style Key] = '010EE2I306'
```

###
###
###
###
###
###
###
###
###
###
###
