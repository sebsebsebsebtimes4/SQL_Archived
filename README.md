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

### Check_NSLS_EOH_Replen_days

```
with nsls as
(
SELECT 
    --  [Style]
    -- ,[Colour]
    -- ,[Length]
    -- ,[Size]
    CONCAT([Style],[Colour],[Size],[Length]) as Article
      ,[Calday]
      ,[NSLS]
      ,[EOH]
      ,[Replenish]
    
  FROM [DataLake].[Dist].[Movement]
  Where Store in ('1144') and EOH <= 0  and Calday > '2023-01-01' -- and NSLS >0
),

replen as
(
SELECT 
 
	  CONCAT([Style],[Colour],[Size],[Length]) as 'RE-Article'
      ,[Calday]
      ,[NSLS]
      
      ,[Replenish]
	  ,[EOH]
    
  FROM [DataLake].[Dist].[Movement]
  Where Store in ('1144')  and Calday > '2023-01-01' and Replenish >0  and CONCAT([Style],[Colour],[Size],[Length]) in ('881LA1319961239-42')
)

Select * 
,(
  select TOP(1) 
         replen.Calday 
		 from replen
         where replen.[RE-Article] = nsls.Article and
               replen.Calday > nsls.Calday
		 order by replen.Calday Asc
) as Replen_Date
  ,DATEDIFF(day,[Calday],(
  select TOP(1) 
         replen.Calday 
	 from replen
         where replen.[RE-Article] = nsls.Article and
               replen.Calday > nsls.Calday
	 order by replen.Calday Asc
  )) as 'Days to Receive'

from nsls
where Article in ('881LA1319961239-42') order by Calday
```
### Check_Orders_2_Screens

```
declare @slsorder as varchar(10)
set @slsorder='0133729706'

SELECT [SalesDocument] as 'SalesOrder'
      ,[Ship-toparty]
	  ,CreatedOnDate
      ,[Corrqty]
	  ,sum([Corrqty])over() as 'totalCorrqty'
      ,[ConfirmedQty]
	  ,sum([ConfirmedQty])over() as 'totalConfirmedQTY'

FROM [DataLake].[AFS].[SalesOrder]
where SalesDocument in (@slsorder)

SELECT 
       Referencedoc as 'SalesOrder'
	  ,[SalesDocument] as 'DeliveryOrder'
	  ,Deliveryqty
	  ,sum([Deliveryqty]) over() as 'TotalDELqty'
	  ,ActualGIdate
  FROM [DataLake].[AFS].[DeliveryItem]
  where Referencedoc in (@slsorder)
```
### Checking_SLS_Replen_Days

```
with replen as (
SELECT [Store]
      ,[Style]
      ,[Colour]
      ,[Length]
      ,[Size]
      ,[Calday]
	  
   --   ,[NREC]
   --   ,[NSLS]
   --   ,[EOH]
   --   ,[Initial]
      ,[Replenish]
     
  FROM [DataLake].[Dist].[Movement]
  Where Store in ('1144') and Calday > '2023-01-01' and Replenish > 0-- and Style in ('013EE2K326') and Colour in ('044') and Size in ('M') 

),

loc as (
SELECT [Store]
      ,[StoreName]
	  ,[Country]
      ,[SalesOrganization]
      ,[DistributionChannel]
FROM [DataLake].[Retail].[Store]
),

cal as (
SELECT [Date]
      ,[ISODate]
      ,[Year]
      ,[Month]
      ,[AddMonthName]
      ,[DayOfWeek]
      ,[DayOfWeekName]
      ,[WeekNumber]
  FROM [DataLake].[master].[Calendar]
  where year = '2023'
),

sty as (
SELECT [Style]
      ,[BrandText]
      ,[BrandTextShort]
      ,[DivisionSet]
    --  ,[DivisionSetText]
    --  ,[GenderDescription]
      ,[MaterialGroup]
	--  ,max([CostPriceEUR])
   
  FROM [Reporting].[MasterDataViews].[ProductStyle]
  group by [Style],[BrandText],[BrandTextShort],[DivisionSet],[MaterialGroup]
)


SELECT nsl.[Store]
      ,nsl.[Style]
      ,nsl.[Colour]
	  ,nsl.[Size]
      ,nsl.[Length]
      ,nsl.[Calday] as NSLS_Date
      ,nsl.[NSLS]
	  ,nsl.EOH
	  ,replen.Calday as Replen_Date
	  ,replen.Replenish
	  ,DATEDIFF(day,nsl.[Calday],replen.Calday) as 'Days-Between'
      ,ROW_NUMBER() Over (Partition by Concat(nsl.[Store],nsl.[Style],nsl.[Colour],nsl.[Size],nsl.[Length],nsl.[Calday]) order by nsl.[Calday]) as 'Rank1'
--	  ,ROW_NUMBER() Over (Partition by Concat(nsl.[Store],nsl.[Style],nsl.[Colour],nsl.[Size],nsl.[Length],nsl.[Calday]) order by replen.Calday ) as 'Rank2' -- some problems here

	  ,loc.[Country]
	  ,[DayOfWeekName] as 'Replen_Day'

	  ,[MaterialGroup]
	
      
  FROM [DataLake].[Dist].[Movement] as nsl

  left join replen on 
      nsl.Store = replen.Store 
  and nsl.Style = replen.Style
  and nsl.Colour = replen.Colour
  and nsl.Size = replen.Size
  and nsl.Length = replen.Length
  and replen.Calday > nsl.Calday

  left join loc on nsl.Store = loc.Store
  left join cal on replen.Calday = cal.Date
  left join sty on nsl.Style = sty.Style
     
  Where  nsl.Calday >= '2023-01-01' and nsl.NSLS > 0 and nsl.EOH <= 0  and nsl.Store in ('1144')  and nsl.Colour in ('285')  and nsl.Style in ('013EE1B327') 

```

### DC_Location_By_Date
### Declare_Date
### Find_Columns_Tables
### IRP_Order_QTY
### Join_itsself
### Next_Arrive_Replen_Del
### Option_MKD_FP
### Option_Price
### Option_Price_Retail
### Sales_Order_by_Week_Country
### Sales_order_process_days
### Sales_order_to_DEL
### Sales_Pcs_By_Store
### Set_Week_Year

