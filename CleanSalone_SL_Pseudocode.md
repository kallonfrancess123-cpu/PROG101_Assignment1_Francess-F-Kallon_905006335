# CleanSalone SL - Pseudocode

**Project:** Community Waste Reporting and Collection Prioritisation System

This is language-independent pseudocode for the PROG101 logical design assignment. It is not executable source code. The numeric thresholds below are illustrative pilot parameters and must be approved before real deployment.

```text
CONSTANT DUPLICATE_WINDOW_HOURS = 72
CONSTANT HIGH_SCORE_MIN = 6
CONSTANT MEDIUM_SCORE_MIN = 3
CONSTANT FOLLOW_UP_DAYS = 3

PROCEDURE SubmitWasteReport()
    INPUT council, ward, locationDescription, wasteCategory
    INPUT volumeEstimate, affectsDrainOrWaterway
    INPUT blocksRoadOrPublicArea, nearSensitiveSite
    INPUT reporterContact OPTIONAL

    WHILE RequiredFieldsValid(council, ward, locationDescription,
                              wasteCategory, volumeEstimate) = FALSE
          AND userHasNotCancelled = TRUE
        DISPLAY "Complete the required fields or cancel."
        INPUT corrected report details
    END WHILE

    IF userHasCancelled = TRUE THEN
        DISPLAY "Report not submitted."
        RETURN
    END IF

    existingCase = FindOpenDuplicate(council, ward,
                     locationDescription, wasteCategory,
                     CURRENT_DATETIME - 72 HOURS)

    IF existingCase IS NOT NULL THEN
        AttachAdditionalDetails(existingCase.caseID, reportDetails)
        caseID = existingCase.caseID
        priority = RecalculatePriority(existingCase, reportDetails, CURRENT_DATETIME)
        IF priority = "SPECIALIST ESCALATION" THEN
            assignedRoute = LookupSpecialistRoute(council)
            IF assignedRoute = NOT_FOUND THEN
                status = "NEEDS MANUAL SPECIALIST ROUTING"
            ELSE
                status = "ESCALATED FOR SPECIALIST REVIEW"
            END IF
        ELSE
            assignedRoute = LookupApprovedRoute(council, ward)
            IF assignedRoute = NOT_FOUND THEN
                status = "NEEDS MANUAL ROUTING"
            ELSE
                status = existingCase.status
            END IF
        END IF
        SaveUpdate(caseID, priority, assignedRoute, status, CURRENT_DATETIME)
    ELSE
        caseID = GenerateUniqueCaseID()

        IF wasteCategory = "MEDICAL" OR wasteCategory = "HAZARDOUS" THEN
            priority = "SPECIALIST ESCALATION"
            assignedRoute = LookupSpecialistRoute(council)
            IF assignedRoute = NOT_FOUND THEN
                status = "NEEDS MANUAL SPECIALIST ROUTING"
            ELSE
                status = "ESCALATED FOR SPECIALIST REVIEW"
            END IF
        ELSE
            score = CalculateRiskScore(reportDetails, 0)
            priority = ClassifyScore(score)
            assignedRoute = LookupApprovedRoute(council, ward)

            IF assignedRoute = NOT_FOUND THEN
                status = "NEEDS MANUAL ROUTING"
            ELSE
                status = "VALIDATED"
            END IF
        END IF

        SaveNewReport(caseID, reportDetails, priority,
                      assignedRoute, status, CURRENT_DATETIME)
    END IF

    DISPLAY caseID, priority, status
    DISPLAY "Collection timing must be confirmed by the responsible service."
END PROCEDURE

FUNCTION CalculateRiskScore(report, ageDays)
    score = 0
    IF report.affectsDrainOrWaterway = TRUE THEN score = score + 3
    IF report.blocksRoadOrPublicArea = TRUE THEN score = score + 2
    IF report.volumeEstimate = "LARGE" THEN score = score + 2
    IF report.nearSensitiveSite = TRUE THEN score = score + 2
    IF report.wasteCategory = "ILLEGAL DUMPING" OR
       report.wasteCategory = "OPEN BURNING" THEN score = score + 2
    IF ageDays >= FOLLOW_UP_DAYS THEN score = score + 1
    RETURN score
END FUNCTION

FUNCTION ClassifyScore(score)
    IF score >= HIGH_SCORE_MIN THEN
        RETURN "HIGH"
    ELSE IF score >= MEDIUM_SCORE_MIN THEN
        RETURN "MEDIUM"
    ELSE
        RETURN "ROUTINE"
    END IF
END FUNCTION

FUNCTION RecalculatePriority(existingCase, newDetails, currentTime)
    mergedReport = MergeReportDetails(existingCase, newDetails)
    IF mergedReport.wasteCategory = "MEDICAL" OR
       mergedReport.wasteCategory = "HAZARDOUS" THEN
        RETURN "SPECIALIST ESCALATION"
    END IF
    ageDays = DaysBetween(existingCase.submittedAt, currentTime)
    RETURN ClassifyScore(CalculateRiskScore(mergedReport, ageDays))
END FUNCTION

PROCEDURE ReviewOpenReports()
    FOR EACH report IN OpenReports
        IF report.wasteCategory = "MEDICAL" OR
           report.wasteCategory = "HAZARDOUS" THEN
            EnsureSpecialistReferral(report.caseID)
        ELSE
            ageDays = DaysBetween(report.submittedAt, CURRENT_DATETIME)
            score = CalculateRiskScore(report, ageDays)
            report.priority = ClassifyScore(score)
        END IF

        IF DaysSinceLastUpdate(report.caseID) >= FOLLOW_UP_DAYS THEN
            AddToFollowUpQueue(report.caseID)
        END IF
    END FOR

    SortQueueBySpecialistThenPriorityThenOldestFirst(OpenReports)
END PROCEDURE
```

## Priority Rule Summary

- **SPECIALIST ESCALATION:** medical or hazardous waste; bypasses the ordinary numeric score.
- **HIGH:** score is 6 or more.
- **MEDIUM:** score is 3 to 5.
- **ROUTINE:** score is 0 to 2.

## Sample Score

A large waste pile beside a market that affects a drain receives 3 points for drain/waterway impact, 2 for large volume and 2 for proximity to a sensitive site. Total = 7, so the initial label is **HIGH**.
