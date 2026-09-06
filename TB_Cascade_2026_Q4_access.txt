SELECT
    V.PatientID,
    V.VisitTypeCode,
    (SELECT TOP 1 HFRCode FROM tblConfig) AS HFRCode,
    V.NumDaysDispensed,
    V.TBScreeningIDSameDayScreening,
    V.Weight,
    V.TBScreeningID,
    V.VisitDate AS ScreenedDate,
    P.Sex,
    IIf(UCase(Trim(V.NowBreastfeeding)) = "YES", 1, 0) AS Breastfeeding,
    IIf(
        V.TBScreeningID In
            ("CXR+", "GX+ve", "SS+", "TB LAM +ve", "Ped SC+ve", "TB SC +ve", "Conf/Yes")
        Or V.TBScreeningIDSameDayScreening In
            ("CXR+", "GX+ve", "SS+", "TB LAM +ve", "Ped SC+ve", "TB SC +ve", "Conf/Yes")
        Or TP.PatientID Is Not Null,
        "POS",
        Null
    ) AS TB_POS,
    IIf(
        V.TBScreeningID In
            ("CXR+", "GX+ve", "SS+", "TB LAM +ve", "Ped SC+ve", "TB SC +ve", "Conf/Yes")
        Or V.TBScreeningIDSameDayScreening In
            ("CXR+", "GX+ve", "SS+", "TB LAM +ve", "Ped SC+ve", "TB SC +ve", "Conf/Yes")
        Or TP.PatientID Is Not Null,
        IIf(
            Exists
                (SELECT *
                 FROM tblVisits AS VT
                 WHERE VT.PatientID = V.PatientID
                   AND VT.VisitDate >= V.VisitDate
                   AND
                       (VT.TBRXID In ("START TB", "CTN TB", "CPLT TB", "STOP TB")
                        Or VT.TBRXIPTID In ("START TB", "CTN TB", "CPLT TB", "STOP TB")
                        Or VT.AntiTBStartDate Is Not Null
                        Or VT.NoDaysTBDrugsDispensed > 0)
                ),
            1,
            0
        ),
        Null
    ) AS POS_Started_TB_Treatment,
    A.DateStartART
FROM
    ((tblVisits AS V
       INNER JOIN tblPatients AS P
         ON V.PatientID = P.PatientID)
     LEFT JOIN
        (SELECT DISTINCT
             T.PatientID,
             IIf(T.RefVisitDate Is Null, T.TestDate, T.RefVisitDate) AS ScreenedDate
         FROM tblTests AS T
         WHERE T.TestTypeID In ("CXR", "GX", "SS", "LAM")
           AND
               (T.ResultNumeric = 1
                Or UCase(Trim(T.ResultNotes)) In ("POS", "POSITIVE", "DETECTED"))
        ) AS TP
       ON V.PatientID = TP.PatientID
      AND V.VisitDate = TP.ScreenedDate)
    LEFT JOIN
        (SELECT X.PatientID, Min(X.DateStartART) AS DateStartART
         FROM
             (SELECT PatientID, VisitDate AS DateStartART
              FROM tblVisits
              WHERE ARVStatusCode = 2

              UNION ALL

              SELECT PatientID, DateStartARTAtAnotherClinic
              FROM tblStartARTanotherClinic
              WHERE DateStartARTAtAnotherClinic Is Not Null

              UNION ALL

              SELECT PatientID, DateStartART
              FROM tblTreatmentBaselineValues
              WHERE DateStartART Is Not Null
             ) AS X
         GROUP BY X.PatientID
        ) AS A
      ON V.PatientID = A.PatientID
WHERE V.VisitDate >= #07/01/2026#
  AND V.VisitDate <  #10/01/2026#
ORDER BY V.VisitDate, V.PatientID;
