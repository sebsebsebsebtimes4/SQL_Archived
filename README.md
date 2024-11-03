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
where concat(right(m.[IRP CalYMth],4),'-',left(m.[IRP CalYMth],2)) >= @StartIRP and [Style Key] = '010EE2I306'
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
,
(
select TOP(1) 
       replen.Calday 
from replen
where replen.[RE-Article] = nsls.Article and replen.Calday > nsls.Calday
order by replen.Calday Asc

) as Replen_Date
,DATEDIFF(day,[Calday]
,(
select TOP(1) 
       replen.Calday 
from replen
where replen.[RE-Article] = nsls.Article and replen.Calday > nsls.Calday
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
    --,[NREC]
    --,[NSLS]
    --,[EOH]
    --,[Initial]
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
    --,[DivisionSetText]
    --,[GenderDescription]
      ,[MaterialGroup]
    --,max([CostPriceEUR])

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
    --,ROW_NUMBER() Over (Partition by Concat(nsl.[Store],nsl.[Style],nsl.[Colour],nsl.[Size],nsl.[Length],nsl.[Calday]) order by replen.Calday ) as 'Rank2' -- some problems here
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

### DC_LocationEOH_By_Date
```
with calen as (
SELECT [CalDay]
      ,[DayName]
      ,[Year]
      ,[WeekOfYearName]

FROM [Reporting].[RetailViews].[Calendar]
where [DayName] in ('Sunday') and [Year]>'2022'
)

SELECT [CalendarDay]
      ,[DayName]
      ,[WeekOfYearName]
      ,[Material]
      ,[Color]
      ,[Plant]
      ,case 
      when Plant = 'DE01' then 'FIEGE IBBENBUREN'
      when Plant = 'DE02' then 'RETURNS'
      when Plant = 'DE04' then 'RETURNS'
      when Plant = 'DE05' then 'SWISS RETURNS'
      when Plant = 'DE08' then 'DC EUROPE'
      when Plant = 'DE11' then 'DC DC OSNABBRUCK'
      when Plant = 'DE15' then 'DC DC OSNABBRUCK'
      when Plant = 'DE99' then 'CORSINA'
      when Plant = 'US08' then 'DC US'
      else 'UNKNOWN'
      end as 'Plant_Name'
      ,[StorageLocation]
      ,case 
      when [StorageLocation] = '1000' then 'DCE Warehouse'
      when [StorageLocation] = '1001' then 'DCE QC'
      when [StorageLocation] = '2000' then 'DCE IBB Wareh'
      when [StorageLocation] = '2001' then 'DCE IBB QC Area'
      when [StorageLocation] = '4000' then 'DCE Outlet'
      else 'UNKNOWN'
      end as 'Location_Name'
      ,[Quantity]

FROM [Reporting].[SupplyChainViews].[MD04withHistory] as main
left join calen on main.CalendarDay = calen.CalDay
Where [DayName] is not NULL and CalendarDay > GETDATE()-60

```
### Declare_Date
```
DECLARE @MostRecentMonday as datetime = DATEDIFF(day,
                                                 0, 
						 GETDATE() - DATEDIFF(day, 0, GETDATE()) %7
)

DECLARE @MostRecentMonday1 as varchar(10) = convert(date, @MostRecentMonday)

PRINT @MostRecentMonday1
--Calculating Previous Sunday
--DECLARE @CurrentWeekday INT = DATEPART(WEEKDAY, GETDATE())
--DECLARE @LastSunday DATETIME = DATEADD(day, -1 *(( @CurrentWeekday % 7) - 1), GETDATE())
--PRINT @LastSunday
select GETDATE() - DATEDIFF(day, 0, GETDATE())%7

```

### Find_Columns_Tables
```
SELECT c.name  AS 'ColumnName'
       ,t.name AS 'TableName'
FROM sys.columns c
JOIN sys.tables  t
     ON c.object_id = t.object_id
WHERE c.name LIKE '%EES%'
ORDER BY TableName
        ,ColumnName;

```

### IRP_Order_QTY
```
SELECT CONCAT(LEFT([InitialRequirementPeriod],4),'-',right([InitialRequirementPeriod],2)) as InitialRequirementPeriod
       ,[DivisionSet]
       ,[ProductClass (nc)]
       ,a.[Style]
       ,plant
       ,sum([PUR(PCS)]) as 'PUR(PCS)'
       ,sum([PUR(PCS)AB]) as 'PUR(PCS)AB'
       ,sum([REC(PCS)]) as 'REC(PCS)'
FROM [Reporting].[SupplyChainViews].[PurchasingOrderConfirmationShipment] as a
     left join [Reporting].[MasterDataViews].[ProductStyle] as b
     on a.Style = b.Style 
where [InitialRequirementPeriod] in (202112,202201,202202,202203)
group by CONCAT(LEFT([InitialRequirementPeriod],4),'-',right([InitialRequirementPeriod],2)),[DivisionSet],[ProductClass (nc)],a.[Style],plant
order by CONCAT(LEFT([InitialRequirementPeriod],4),'-',right([InitialRequirementPeriod],2))

```

### Join_itsself

```
--select Store
--FROM [DataLake].[Dist].[Movement] AS E
--inner join 
--(
--Select Store 
--,MAX([Calday]) as maxx
--from [DataLake].[Dist].[Movement]
--Group by Store) AS M
--on E.Store = M.Store
--And E.Calday = M.maxx
--Where E.Store = '1244'

select *

FROM [DataLake].[Dist].[Movement] AS E
inner join 
(
Select Store
       ,MAX([Calday]) as maxx
from [DataLake].[Dist].[Movement]
Group by Store
) AS M

on E.Store = M.Store
And E.Calday = M.maxx


--SELECT v.*
--FROM views AS v
--  JOIN
--    ( SELECT userid, MIN(id) AS entryid
--      FROM views
--      GROUP BY userid
--    ) AS vm
--    ON  vm.userid = v.userid 
--    AND vm.entryid = v.id ;

```
### Next_Arrive_Replen_Del

```
with loc as (
SELECT [Store]
    --,[StoreName]
      ,[Country]
    --,[SalesOrganization]
      ,[DistributionChannel]
FROM [DataLake].[Retail].[Store]
),

del as (
SELECT  RIGHT([Ship-toparty],4) as 'Ship-To-Store' 
       ,[Material]
       ,[Custmaterial]
       ,[EANUPC]
       ,[Referencedoc] as SalesDocumentID
       ,[Createdon]
       ,[SalesDocument] as 'DeliveryDocumentID'
       ,[ActualGIdate]
       ,[DeliveryType]
    -- ,[DeliveryDate]
       ,[Route]
       ,[SalesOrg]
    -- ,[ShippingPoint]
    -- ,[CredContrArea]
    -- ,[Route2]
    -- ,[Changedon]
       ,[Deliveryqty]
       ,[ShippingPoint]

FROM [DataLake].[AFS].[DeliveryItem] 
where left (Material, 3) in ('053') and Deliveryqty > 0 and left(Createdon,4) in ('2023')
),

sty as (
SELECT [Style]
    --,[BrandText]
    --,[BrandTextShort]
      ,[DivisionSet]
    --,[DivisionSetText]
    --,[GenderDescription]
      ,[MaterialGroup]
    --,max([CostPriceEUR])
   
FROM [Reporting].[MasterDataViews].[ProductStyle]
group by [Style],[BrandText],[BrandTextShort],[DivisionSet],[MaterialGroup]
),

replen as (
SELECT [Store]
      ,Concat([Style],[Colour],[Size],[Length]) as 'Article'
      ,[Calday]
      ,[Replenish]

FROM [DataLake].[Dist].[Movement]
where Replenish >0 and LEFT(Style,3) in ('053') and left (Calday,4) in ('2023')
)

Select * ,
         DATEDIFF(day, Createdon,ActualGIdate) as 'DC Process Days',
	 (select TOP(1) 
                 replen.Calday 
	  from replen
          where replen.Article = del.Custmaterial and
		replen.Store = del.[Ship-To-Store] and
                replen.Calday > del.ActualGIdate
          order by replen.Calday Asc
      )
as Replen_Date

from del
left join loc on loc.Store = del.[Ship-To-Store]
left join sty on sty.Style = del.Material

where Country in ('NL','FR','AT','DE') and
                 DistributionChannel in ('30') and
                 ShippingPoint in ('1801') and Custmaterial <>''
order by [Ship-To-Store], Createdon

```
### Option_MKD_FP

```
SELECT 
       [Style] + [Colour] + [Length] as Opt,
       [EOH],
	
       CASE 
         WHEN mkd.ctp <> mkd.upr THEN 'HMK' 
         ELSE 'FP' 
       END AS MKD_Flag
	    
FROM [DataLake].[Dist].[EOH_LastSunday] as E
left join
(
SELECT DISTINCT [style] + [colour] + [Length]  AS Opt, 
                [country], 
                [validfrom], 
                [validto], 
                [pri_upr_inclvat_bc] AS UPR, 
                [pri_ctp_inclvat_bc] AS CTP 
                              
FROM [DataLake].[BI].[OWL_Prices_Retail] as BI 
inner join dist.Calendar as C
ON c.calday >= BI.[validfrom] 
   AND c.calday < BI.[validto]

WHERE  [country] = 'DE'AND [distributionchannel] = '30' and c.CalDay = '2020-07-12'
)
as mkd
on E.Style+E.Colour+E.Length = mkd.Opt

Where Store in ('1898','1827','1191','1818','1008',
'1147','1815','1088','1802','1043','1846','1144',
'1143','1364','1876','1830','1813','1844','1851',
'1368','1190','1001','1805','1197','1198','1042',
'1881','1071','1868','1011','1176','1183','1067',
'1366','1182','1161','1048','1068') and EOH>0 and [Style] + [Colour] + [Length] like '997CC1K816%'

```

### Option_Price

```
SELECT DISTINCT(P.[Style])
       ,(P.[Style]+p.[Colour]) as 'Option'
       ,max([ValidFrom]) as ValidFrom
       ,Max([ValidTo]) as ValidTo
       ,Max([PRI_UPR_InclVAT_BC]) as UPR
       ,Min([PRI_CTP_InclVAT_BC]) as CTP
       ,min([PRI_LLC_InclVAT_BC]) as LLC
       ,min(S.ProductCluster) as ProductCluster
       ,min(ProductClusterText) as ProductClusterText
       ,min(m.[ProductClass]) as ProductClass
       ,min(left(P.[Style],2)) as SSN

FROM [DataLake].[BI].[OWL_Prices_Retail] as P
inner join [DataLake].[Dist].[Calendar] as D
      ON D.calday >= P.[validfrom] 
      AND D.calday < P.[validto] 

left join [DataLake].[BI].[Style] as S
     on P.Style = S.Style

left join [DataLake].[BI].[ProductClusters] as C
     on S.ProductCluster = C.ProductCluster

left join [DataLake].[WSSI_IF].[ADB_PRODUCT] as W
     on P.[Style]+P.[Colour] = w.[Option]

left join [DataLake].[master].[style] as M
     on P.[Style] = M.Style

Where Country = 'DE' and D.CalDay = '2020-11-23' and left(P.[Style],2) = '99'
group by P.Style,P.[Style]+p.[Colour]
order by P.Style,P.[Style]+p.[Colour]


```

### Option_Price_Retail

```
SELECT p.Style
      ,max([ValidFrom]) as ValidFrom
      ,Max([ValidTo]) as ValidTo
      ,min([PRI_UPR_InclVAT_BC]) as RetailUPR
      ,Min([PRI_CTP_InclVAT_BC]) as RetailCTP
    --,Min(p.[PRI_UPR_InclVAT_BC]) as EcomUPR
    --,Min(p.[PRI_CTP_InclVAT_BC]) as EcomCTP
      ,min([PRI_LLC_InclVAT_BC]) as LLC
      ,min(S.ProductCluster) as ProductCluster
      ,min(ProductClusterText) as ProductClusterText
      ,min(m.[ProductClass]) as ProductClass
      ,min([Season Start]) as SSN

FROM [DataLake].[BI].[OWL_Prices_Retail] as P
inner join [DataLake].[Dist].[Calendar] as D
      ON D.calday >= P.[validfrom] 
      AND D.calday < P.[validto] 
  
left join [DataLake].[BI].[Style] as S
     on P.Style = S.Style
  
left join [DataLake].[BI].[ProductClusters] as C
     on S.ProductCluster = C.ProductCluster
  
left join [DataLake].[WSSI_IF].[ADB_PRODUCT] as W
     on P.[Style]+P.[Colour] = w.[Option]
  
left join [DataLake].[master].[style] as M
     on P.[Style] = M.Style

  --left join
  --(SELECT 
  -- distinct(P.style)
  -- ,min(p.[PRI_UPR_InclVAT_BC]) as EcomUPR
  -- ,min(p.[PRI_CTP_InclVAT_BC]) as EcomCTP
  -- from [DataLake].[BI].[OWL_Prices_ecom] as P
  -- inner join [DataLake].[Dist].[Calendar] as D
  -- ON D.calday >= P.[validfrom] 
  -- AND D.calday < P.[validto] 

  -- Where Country ='DE' and D.calday ='2020-10-07'
  -- group by P.Style)
  -- as E
  -- on P.Style = E.Style 
    
Where Country = 'DE' and D.CalDay = '2020-10-07'
group by P.[Style]

```

### Sales_Order_by_Week_Country

```
with main as (
select [SalesDocument]
      ,[CreatedOnDate]
      ,[DocumentDate]
      ,[Ship-toparty]
      ,right([Ship-toparty],4) as 'Store'
      ,[MaterialGroup]
      ,[Material]
      ,[Color]
      ,CONCAT([Material],Color) as 'SOption'
      ,[Corrqty]
      ,[ConfirmedQty]
      ,[DeliveryDate]
      ,[RejectionReason]
      ,[Size]
      ,[Size1]
      ,[Size2]
      ,[EANUPC]
      ,[SalesDocType]
      ,[SalesOrg]
      ,[DistrChannel]
      ,[EAN11]
      ,[ShippingPoint]
      ,[Plant]

FROM [DataLake].[AFS].[SalesOrder]
where DocumentDate between '2023-11-27' and '2023-12-03' and Corrqty > 0
),

loc as (
SELECT [Store]
      ,[StoreName]
      ,[Country]
      ,[SalesOrganization]
      ,[DistributionChannel]
      ,Region
      ,RegionText
      ,CASE
WHEN DistributionChannel = 30 THEN 'Retail'
WHEN DistributionChannel = 31 THEN 'Ecom'
     ELSE 'Outlet'
     END as 'Channel'
FROM [DataLake].[Retail].[Store]
),

calen as (

SELECT [CalDay]
      ,[DayName]
      ,[Year]
      ,[IsoWeek]
      ,[WeekOfYearName]
 FROM [Reporting].[RetailViews].[Calendar]
 ),

 info as
(
 SELECT left(Article,13) as 'SOption'
       ,[GenderDescription]
       ,MaterialGroup
       ,[ProductClusterDescription]
       ,[NOOSEndSeason]

FROM [Reporting].[MasterDataViews].[ProductArticle]
group by left(Article,13),[GenderDescription],MaterialGroup,[ProductClusterDescription],[NOOSEndSeason]
)

select *
   
from main 
left join loc on main.Store = loc.Store
left join calen on main.CreatedOnDate = calen.CalDay
left join info on main.SOption = info.SOption

where Channel in ('Retail','Ecom')

```



### Sales_order_process_days

```
with loc as (
SELECT [Store]
    --,[StoreName]
      ,[Country]
    --,[SalesOrganization]
      ,[DistributionChannel]
FROM [DataLake].[Retail].[Store]
),

del as (
SELECT RIGHT([Ship-toparty],4) as 'Ship-To-Store' 
      ,[Material]
      ,[Custmaterial]
      ,[EANUPC]
      ,[Referencedoc] as SalesDocumentID
      ,[Createdon]
      ,[SalesDocument] as 'DeliveryDocumentID'
      ,[ActualGIdate]
      ,[DeliveryType]
   -- ,[DeliveryDate]
      ,[Route]
      ,[SalesOrg]
   -- ,[ShippingPoint]
   -- ,[CredContrArea]
   -- ,[Route2]
   -- ,[Changedon]
      ,[Deliveryqty]
      ,[ShippingPoint]

FROM [DataLake].[AFS].[DeliveryItem] 
where left (Material, 3) in ('053') and Deliveryqty > 0 and left(Createdon,4) in ('2023')
),

sty as
(
SELECT [Style]
    --,[BrandText]
    --,[BrandTextShort]
      ,[DivisionSet]
    --,[DivisionSetText]
    --,[GenderDescription]
      ,[MaterialGroup]
    --,max([CostPriceEUR])
   
FROM [Reporting].[MasterDataViews].[ProductStyle]
group by [Style],[BrandText],[BrandTextShort],[DivisionSet],[MaterialGroup]
),

replen as
(
SELECT [Store]
      ,Concat([Style],[Colour],[Size],[Length]) as 'Article'
      ,[Calday]
      ,[Replenish]
FROM [DataLake].[Dist].[Movement]
where Replenish >0 and LEFT(Style,3) in ('053') and left (Calday,4) in ('2023')
)

Select * ,
       DATEDIFF(day, Createdon,ActualGIdate) as 'DC Process Days',
       (
         select TOP(1) replen.Calday 
         from replen
         where replen.Article = del.Custmaterial and
               replen.Store = del.[Ship-To-Store] and
               replen.Calday > del.ActualGIdate
	 order by replen.Calday Asc
      ) as Replen_Date

from del
left join loc on loc.Store = del.[Ship-To-Store]
left join sty on sty.Style = del.Material

where Country in ('NL','FR') and DistributionChannel in ('30') and ShippingPoint in ('1801') and Custmaterial <>''
order by [Ship-To-Store], Createdon

```
### Sales_order_to_DEL
```
declare @TY as varchar(10)
set @TY= DATEPART(YEAR,GETDATE())

declare @wk1 as varchar(10)
set @wk1= DATEPART(WEEK,GETDATE())-1

declare @TY1 as varchar(10)
set @TY1 = Concat(@wk1,'.',@TY);

with main as (
select [SalesDocument]
      ,[CreatedOnDate]
      ,[DocumentDate]
      ,[Ship-toparty]
      ,right([Ship-toparty],4) as 'Store'
      ,[MaterialGroup]
      ,[Material]
      ,[Color]
      ,CONCAT([Material],Color) as 'SOption'
      ,[Corrqty] as 'Intend Allocation'
      ,[ConfirmedQty] as 'Final Confirmed Unit'
      ,[DeliveryDate]
      ,[RejectionReason]
      ,[Size]
      ,[Size1]
      ,[Size2]
      ,[EANUPC]
      ,[SalesDocType]
      ,[SalesOrg]
      ,[DistrChannel]
      ,[EAN11]
      ,[ShippingPoint]
      ,[Plant]
FROM [DataLake].[AFS].[SalesOrder]
where DocumentDate between '2023-11-27' and getdate() and Corrqty > 0
),

loc as (
SELECT [Store]
      ,[StoreName]
      ,[Country]
      ,[SalesOrganization]
      ,[DistributionChannel]
      ,Region
      ,RegionText
      ,CASE
       WHEN DistributionChannel = 30 THEN 'Retail'
       WHEN DistributionChannel = 31 THEN 'Ecom'
       ELSE 'Outlet'
       END as 'Channel'
FROM [DataLake].[Retail].[Store]
),

calen as (
SELECT [CalDay]
      ,[DayName]
      ,[Year]
      ,[IsoWeek]
      ,[WeekOfYearName]
 FROM [Reporting].[RetailViews].[Calendar]
 ),

delivery as (
SELECT Referencedoc as 'SalesOrder'
      ,EANUPC 
      ,sum(Deliveryqty) as DeliveryQTY
FROM [DataLake].[AFS].[DeliveryItem]
group by Referencedoc,EANUPC
),

info as (
SELECT left(Article,13) as 'SOption'
      ,[GenderDescription]
      ,MaterialGroup
      ,[ProductClusterDescription]
      ,[NOOSEndSeason]
      ,[SeasonYear]
      ,[SeasonCode]
  
FROM [Reporting].[MasterDataViews].[ProductArticle]
group by left(Article,13),[GenderDescription],MaterialGroup,[ProductClusterDescription],[NOOSEndSeason],[SeasonYear],[SeasonCode]
)

select *
   
from main 

left join loc on main.Store = loc.Store
left join calen on main.CreatedOnDate = calen.CalDay
left join info on main.SOption = info.SOption
left join delivery on main.SalesDocument = delivery.SalesOrder and main.EANUPC = delivery.EANUPC

where Channel in ('Retail','Ecom') and [WeekOfYearName] in (@TY1)

```

### Sales_Pcs_By_Store

```
SELECT [Store]
      ,datepart(week, CalendarDay) as 'Selling Week'
      ,sum([NSLS@PCS]) as NSLS@PCS
      ,sum(ISNULL(UPR*NSLS@PCS, [NSLS@CTP iV BC])) as NSLS@UPRivBC
      ,sum([NSLS@CTP iV BC]) as 'NSLS@CTP iV BC'
      ,sum([NSLS@ASP iV BC]) as 'NSLS@ASP iV BC'
      ,sum([NSLS@LLC BC]) as 'NSLS@LLC BC'
      ,sum(ISNULL(UPR*NSLS@PCS, [NSLS@CTP iV BC]) - [NSLS@CTP iV BC]) as 'MD Value'
      ,sum([NSLS@CTP iV BC] - [NSLS@ASP iV BC]) as 'Promo Value'
      ,sum(ISNULL(UPR*NSLS@PCS, [NSLS@CTP iV BC]) - [NSLS@ASP iV BC]) as 'Discount Value'

      ,CASE
       WHEN ProductClusterText in ('Blouses','Shirts') THEN 'Blouse_Shirt'
       WHEN ProductClusterText in ('Dresses','Overalls','Skirts') THEN 'Dress'
       WHEN ProductClusterText in ('Jackets Indoor/Blazer') THEN 'Indoor'
       WHEN ProductClusterText in ('Coats','Jackets Outdoor','Leather') THEN 'Outerwear'
       WHEN ProductClusterText in ('Pants Denim','Pants Knitted','Pants Woven') THEN 'Pant'
       WHEN ProductClusterText in ('Shorts') THEN 'Short'
       WHEN ProductClusterText in ('Suiting') THEN 'Suit'
       WHEN ProductClusterText in ('Sweaters','Sweatshirts') THEN 'Sweater'
       WHEN ProductClusterText in ('Poloshirts','T-Shirts') THEN 'T'
       ELSE 'Lifestyle' 
       END AS 'Cluster'

FROM [DataLake].[StoreApp].[BonPosition] as M
left join (
        SELECT [Style] 
              ,Max([PRI_UPR_InclVAT_BC]) as UPR
        FROM [DataLake].[BI].[OWL_Prices_Retail]
        Where Country = 'DE'
        Group by Style
          ) as U
     on M.Style = U.Style

left join [DataLake].[BI].[Style] as S
     on M.Style = S.Style

left join [DataLake].[BI].[ProductClusters] as C
     on S.ProductCluster = C.ProductCluster

Where Store in ('1001','1008','1042','1043','1048','1067','1071','1088','1143','1144','1147','1161','1176','1183','1190','1191','1198','1364','1366','1802','1813','1815','1818','1827','1830','1846','1851','1868','1876','1881','1898')
and CalendarDay between '2020-07-20' and '2020-11-22'
group by Store,datepart(week, CalendarDay),
      CASE
       WHEN ProductClusterText in ('Blouses','Shirts') THEN 'Blouse_Shirt'
       WHEN ProductClusterText in ('Dresses','Overalls','Skirts') THEN 'Dress'
       WHEN ProductClusterText in ('Jackets Indoor/Blazer') THEN 'Indoor'
       WHEN ProductClusterText in ('Coats','Jackets Outdoor','Leather') THEN 'Outerwear'
       WHEN ProductClusterText in ('Pants Denim','Pants Knitted','Pants Woven') THEN 'Pant'
       WHEN ProductClusterText in ('Shorts') THEN 'Short'
       WHEN ProductClusterText in ('Suiting') THEN 'Suit'
       WHEN ProductClusterText in ('Sweaters','Sweatshirts') THEN 'Sweater'
       WHEN ProductClusterText in ('Poloshirts','T-Shirts') THEN 'T'
       ELSE 'Lifestyle' end
```

### Set_Week_Year
```
declare @TY as varchar(10)
declare @LY as varchar(10)
declare @wk1 as varchar(10)
declare @wk2 as varchar(10)
declare @wk3 as varchar(10)
declare @wk4 as varchar(10)

declare @TY1 as varchar(10)
declare @TY2 as varchar(10)
declare @TY3 as varchar(10)
declare @TY4 as varchar(10)

declare @LY1 as varchar(10)
declare @LY2 as varchar(10)
declare @LY3 as varchar(10)
declare @LY4 as varchar(10)


set @TY= DATEPART(YEAR,GETDATE())
set @LY= DATEPART(YEAR,GETDATE())-1
set @wk1= DATEPART(WEEK,GETDATE())-1
set @wk2= DATEPART(WEEK,GETDATE())-2
set @wk3= DATEPART(WEEK,GETDATE())-3
set @wk4= DATEPART(WEEK,GETDATE())-4

set @TY1 = Concat(@wk1,'.',@TY)
set @TY2 = Concat(@wk2,'.',@TY)
set @TY3 = Concat(@wk3,'.',@TY)
set @TY4 = Concat(@wk4,'.',@TY)

set @LY1 = Concat(@wk1,'.',@LY)
set @LY2 = Concat(@wk2,'.',@LY)
set @LY3 = Concat(@wk3,'.',@LY)
set @LY4 = Concat(@wk4,'.',@LY)

select @TY1,
       @TY2,
       @TY3,
       @TY4,
       @LY1,
       @LY2,
       @LY3,
       @LY4
```
