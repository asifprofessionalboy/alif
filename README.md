-- Step 1: Calculate Holidays
WITH HolidayData AS (
    SELECT 
        DATEPART(month, Hdate) AS Month,
        DATEPART(year, Hdate) AS Year,
        Location,
        COUNT(*) AS HolidayCount
    FROM App_HolidayMaster
    WHERE Location = 'TSL WORKS O&M'
    GROUP BY DATEPART(month, Hdate), DATEPART(year, Hdate), Location
),

-- Step 2: Precompute Attendance Details
AttendanceData AS (
    SELECT
        MasterID,
        AadharNo,
        VendorCode,
        WorkOrderNo,
        WorkManSl,
        EngagementType,
        DayDef,
        Present,
        Dates,
        SUM(CASE 
                WHEN EngagementType = 'ManPowerSupply' AND Present = 'True' THEN 
                    CASE WHEN DayDef = 'HF' THEN 0.5 ELSE 1.00 END 
                ELSE 0.00 
            END) AS NoOfDaysWorkedMP,
        SUM(CASE 
                WHEN EngagementType = 'ItemRate' AND Present = 'True' THEN 1.00 
                ELSE 0.00 
            END) AS NoOfDaysWorkedRate,
        SUM(CASE 
                WHEN EngagementType = 'ManPowerSupply' AND Present = 'False' THEN 1.00 
                ELSE 0.00 
            END) AS NoOfDaysAbsMP
    FROM App_AttendanceDetails
    GROUP BY MasterID, AadharNo, VendorCode, WorkOrderNo, WorkManSl, EngagementType, DayDef, Present, Dates
),

-- Step 3: Precompute Employee Details
EmployeeData AS (
    SELECT
        ID AS MasterID,
        Name AS WorkManName,
        WorkManCategory,
        PFNo,
        ESINo,
        UANNo,
        EMP_PF_EXEMPTED,
        EMP_ESI_EXEMPTED,
        VendorName,
        LabourState
    FROM App_EmployeeMaster
),

-- Step 4: Precompute Wages Data
WagesData AS (
    SELECT
        MasterID,
        WorkOrderNo,
        WorkManSl,
        MonthWage,
        YearWage,
        LocationCode,
        MAX(CASE WHEN Basic_rate IS NOT NULL THEN Basic_rate ELSE Basic END) AS BasicRate,
        MAX(CASE WHEN Da_rate IS NOT NULL THEN Da_rate ELSE DA END) AS DARate,
        MAX(CASE WHEN OtherAllow IS NOT NULL THEN OtherAllow ELSE Allowance END) AS OtherAllow,
        MAX(CASE WHEN HRA IS NOT NULL THEN HRA ELSE HRA END) AS HRA,
        MAX(PieceRate) AS PieceRate,
        MAX(OverTimeAmt) AS OverTimeAmt,
        MAX(CashPaymentAmt) AS CashPaymentAmt,
        MAX(OtherDeduAmt) AS OtherDeduAmt
    FROM App_WagesDetailsJharkhand
    LEFT JOIN App_EmployeeMaster ON App_WagesDetailsJharkhand.AadharCard = App_EmployeeMaster.AadharCard
    GROUP BY MasterID, WorkOrderNo, WorkManSl, MonthWage, YearWage, LocationCode
),

-- Step 5: Calculate Total Days and Sundays in a Month
MonthData AS (
    SELECT 
        YEAR(Dates) AS YearWage,
        MONTH(Dates) AS MonthWage,
        DAY(EOMONTH(Dates)) AS TotDaysInMonth,
        DATEDIFF(WEEK, DATEADD(MONTH, DATEDIFF(MONTH, 0, Dates), 0), EOMONTH(Dates)) AS TotSunDays
    FROM App_AttendanceDetails
    WHERE Dates BETWEEN '2024-11-01' AND '2024-11-30'
)

-- Final Query: Combine All Precomputed Data
SELECT 
    AD.MasterID,
    AD.AadharNo,
    ED.WorkManName,
    ED.WorkManCategory,
    ED.PFNo,
    ED.ESINo,
    ED.UANNo,
    ED.VendorName,
    ED.LabourState AS StateName,
    AD.WorkOrderNo,
    WD.BasicRate,
    WD.DARate,
    WD.OtherAllow,
    WD.HRA,
    MD.TotDaysInMonth,
    MD.TotSunDays,
    HD.HolidayCount AS TotHoliDays,
    AD.NoOfDaysWorkedMP,
    AD.NoOfDaysWorkedRate,
    AD.NoOfDaysAbsMP
FROM AttendanceData AD
LEFT JOIN EmployeeData ED ON AD.MasterID = ED.MasterID
LEFT JOIN WagesData WD ON AD.MasterID = WD.MasterID AND AD.WorkOrderNo = WD.WorkOrderNo
LEFT JOIN MonthData MD ON YEAR(AD.Dates) = MD.YearWage AND MONTH(AD.Dates) = MD.MonthWage
LEFT JOIN HolidayData HD ON YEAR(AD.Dates) = HD.Year AND MONTH(AD.Dates) = HD.Month
WHERE AD.VendorCode = '15978'
  AND AD.WorkOrderNo IN ('4700026128', '4700022755')
ORDER BY ED.WorkManName;
