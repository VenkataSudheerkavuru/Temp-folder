Step-by-Step Flow (as in code)

1. Calculate Shift Duration

Convert Shift Start and Shift End into seconds.

Compute shiftDiff = shiftEnd - shiftStart.

If shiftDiff < 0, add 86400 seconds (to handle overnight shifts).



2. Handle Break Shift

If shift type is BREAK_SHIFT, calculate break duration (brkDiff = breakIn - breakOut).

If brkDiff < 0, add 86400 to adjust.

Subtract break duration from shiftDiff.



3. Fetch Special Request Types

Call specialRequestTypeRestClient.getSpecialRequestTypes(affiliateId).

Build lists of abbreviations:

Combination abbreviations (like POD, PFH, etc.)

Present abbreviations (P + type, type + P, MS + type, etc.)

Absence abbreviations (A + type, type + A)

LOP abbreviations (LOP + type, type + LOP)


Check if attendanceStatusCode matches any of these lists.



4. Set Shift Hours

If isNormalShiftWithBreakHours and totalBreakHour available →
shiftHours = shiftDiff - breakBetweenBreakInAndOut.

Else if fixedBreakTime available →
shiftHours = shiftDiff - fixedBreakTime.

Else →
shiftHours = shiftDiff.

Save into processingRecord.setShiftHours.



5. Set Required Hours

If attendanceStatusCode is one of (WO|H|WOP|HP|LOP|OH) OR leave applied:

If leave applied:

Full Day Leave + 1st Half Absence → RequiredHours = shiftDiff (minus break).

Full Day Leave without 1st Half Absence → RequiredHours = 00:00:00.

Partial Leave or Special Request Combo → Call updateRequiredHoursForPartialLeaveAndSpecialRequestType.


Else (Holiday, WO, etc.) → RequiredHours = 00:00:00.


Else if Special Request Applied:

Full Day → RequiredHours = shiftDiff (minus break, based on precedence).

Hourly Special Request → RequiredHours = shiftDiff (minus applicable break).

Else:

If Present Code OR Combination Code → RequiredHours = shiftDiff (minus break).

If Absence Code → RequiredHours = shiftDiff (minus break).

If LOP Code → RequiredHours = (shiftDiff - break) / 2.



Else (no leave/special request):

RequiredHours = shiftDiff (minus applicable break).


        List<String> presentAbbreviations = specialRequestTypes.stream().flatMap(requestType -> {
            String abbreviation = requestType.getSpecialRequestAbbreviation();
            return List.of("P" + abbreviation, abbreviation + "P", "MS" + abbreviation, abbreviation + "MS", "M" + abbreviation, abbreviation + "M").stream();
        }).collect(Collectors.toList());

        boolean isValidPresentSpecialRequestAbbreviation = presentAbbreviations.stream().anyMatch(modifiedAbbreviation -> {
            return attendanceStatusCode.equals(modifiedAbbreviation);
        });

        List<String> absenceAbbreviations = specialRequestTypes.stream().flatMap(requestType -> {
            String abbreviation = requestType.getSpecialRequestAbbreviation();
            return List.of("A" + abbreviation, abbreviation + "A").stream();
        }).collect(Collectors.toList());


        boolean isValidAbsenceSpecialRequestAbbreviation = absenceAbbreviations.stream().anyMatch(modifiedAbbreviation -> {
            return attendanceStatusCode.equals(modifiedAbbreviation);
        });

//Get all specialRequestAbbreviation combination of LOP
        List<String> lOPAbbreviations = specialRequestTypes.stream().flatMap(requestType -> {
            String abbreviation = requestType.getSpecialRequestAbbreviation();
            return List.of("LOP" + abbreviation, abbreviation + "LOP").stream();
        }).collect(Collectors.toList());


        boolean isValidLOPSpecialRequestAbbreviation = lOPAbbreviations.stream().anyMatch(modifiedAbbreviation -> {
            return attendanceStatusCode.equals(modifiedAbbreviation);
        });

        SpecialRequestAbbrCombValidityType specialRequestAbbrCombValidityType = prepareSpecialrequestAbbrTypes(isValidPresentSpecialRequestAbbreviation, specialRequestAbbreviation, isValidCombinationSpecialRequestAbbreviation, isValidAbsenceSpecialRequestAbbreviation, isValidLOPSpecialRequestAbbreviation);

        if (shiftDiff < 0) {
            shiftDiff = shiftDiff + 86400;
        }

        if (currentDayShift.getShiftType() == ShiftType.BREAK_SHIFT) {

            long mShiftBreakIN = DateUtil.convertHhMmSsToSeconds(currentDayShift.getShiftTiming().getBreakIn());
            long mShiftBreakOut = DateUtil.convertHhMmSsToSeconds(currentDayShift.getShiftTiming().getBreakOut());

            long brkDiff = (mShiftBreakIN - mShiftBreakOut);
            if (brkDiff < 0) {
                brkDiff = brkDiff + 86400;
            }

            shiftDiff = ((shiftDiff) - (brkDiff));
        }

        //setting values for shift hours
        String breakTimeBasedOnPrecedence = currentDayShift.isNormalShiftWithBreakHours() && processingRecord.getTotalBreakHour() != null ? processingRecord.getTotalBreakHour() : workHourPolicy.getFixedBreakTime();
        Long fixedBreakTimeBtwBreakOutIn = breakHourCalculationService.getFixedBreakTimeBtwBreakInAndOut(currentDayShift.getShiftTiming(), currentDayShift.isNormalShiftWithBreakHours());

        if (breakTimeBasedOnPrecedence == null) {
            processingRecord.setShiftHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff)));
        } else {
            long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(breakTimeBasedOnPrecedence);
            processingRecord.setShiftHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
        }

        // setting values for required hours
        if (processingRecord.getAttendanceStatusCode().matches("WO|H|WOP|HP|LOP|OH") || transactionConfirmation.isLeaveApplied()) {
            if (transactionConfirmation.isLeaveApplied()) {
                if (transactionConfirmation.isIsleaveAppliedForFullDay() && transactionConfirmation.isFirstHalfAbsenceHrs() && transactionConfirmation.isFirstAppliedLeaveFlagInADay() && !transactionConfirmation.isSecondAppliedLeaveFlagInADay()) {
                    LOGGER.info("Inside Full Day MAKING REQUIRED HOURS 0 FOR " + processingRecord.getAccountNo());
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (workHourPolicy.getFixedBreakTime() == null) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                    }
                }

                if (transactionConfirmation.isIsleaveAppliedForFullDay() && !(transactionConfirmation.isFirstHalfAbsenceHrs()) && transactionConfirmation.isFirstAppliedLeaveFlagInADay() && !transactionConfirmation.isSecondAppliedLeaveFlagInADay()) {
                    LOGGER.info("Inside Full Day MAKING REQUIRED HOURS 0 FOR " + processingRecord.getAccountNo());
                    processingRecord.setRequiredHours("00:00:00");
                } else {
                    //this applicable to special request and leave combined case - since special request has no impact in required hours calculation.isSpecial request applied falg is true
                    // leave applied for half day and hourly leave case so isspecial Request applied is passed as false
                    updateRequiredHoursForPartialLeaveAndSpecialRequestType(processingRecord, workHourPolicy, currentDayShift, transactionConfirmation, shiftDiff, fixedBreakTimeBtwBreakOutIn, transactionConfirmation.isSpecialRequestApplied());


                }
            } else {
                processingRecord.setRequiredHours("00:00:00");
            }

        } else if (transactionConfirmation.isSpecialRequestApplied()) {
            if (transactionConfirmation.isSpecialRequestForFullDay()) {
                /*processingRecord.setRequiredHours(workHourPolicy.getMinThresholdWorkHourForFullday());*/
                long fixedBreakTimeInLong = 0L;


                if (specialRequestAbbreviation) {
                    // first precdence should be given to normal shift with  break hours
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (workHourPolicy.getFixedBreakTime() == null) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));
                    } else {

                        if (!CollectionUtils.isEmpty(workHourPolicy.getWorkHourRuleModelList())) {
                            for (int i = 0; i < workHourPolicy.getWorkHourRuleModelList().size(); i++) {
                                if (workHourPolicy.getWorkHourRuleModelList().get(i).isEnableBreakType()) {
                                    List<Long> ids = workHourPolicy.getWorkHourRuleModelList().get(i).getIds();
                                    List<Long> longStream = ids.stream().filter(item -> item.equals(processingRecord.getShiftId())).collect(Collectors.toList());
                                    if (longStream != null) {
                                        fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getWorkHourRuleModelList().get(i).getBreakTime());
                                    }
                                } else {
                                    fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                                }
                            }
                        } else {
                            fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        }
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));

                    }
                }

            } else if (transactionConfirmation.getIsHourlySpecialRequest()) {
                if (currentDayShift.isNormalShiftWithBreakHours()) {
                    processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                } else if (workHourPolicy.getFixedBreakTime() == null) {
                    processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));

                } else {
                    long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                    fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                    processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));

                }
            } else {
                /*processingRecord.setRequiredHours(workHourPolicy.getMinThresholdWorkHourForHalfday());*/
                if ((isValidPresentSpecialRequestAbbreviation || isValidCombinationSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("POD|PFW|PWFH|ODP|FWP|WFHP|ODWFH|WFHOD|ODFW|FWOD|FWWFH|WFHFW|MOD|MFW|MWFH|ODM|FWM|WFHM|MSOD|MSFW|MSWFH|ODMS|FWMS|WFHMS"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (workHourPolicy.getFixedBreakTime() == null) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());

                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                    }
                } else if ((isValidAbsenceSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("AOD|AFW|AWFH|ODA|FWA|WFHA"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (workHourPolicy.getFixedBreakTime() == null) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                    }

                } else if ((isValidLOPSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("LOPFW|FWLOP|ALOP|LOPA|LOPOD|ODLOP|LOPWFH|WFHLOP"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2));
                    } else if (workHourPolicy.getFixedBreakTime() == null) {
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) / 2)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeInLong) / 2));
                    }
                }
            }
        } else {
            //processingRecord.setRequiredHours(workHourPolicy.getMinThresholdWorkHourForFullday());
            if (currentDayShift.isNormalShiftWithBreakHours()) {
                processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
            } else if (workHourPolicy.getFixedBreakTime() == null) {
                processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));

            } else {
                long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                processingRecord.setRequiredHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));

            }
        }

        // Binding data for worked hours
        if (processingRecord.getAttendanceStatusCode().matches("WO|H|LOP|OH") || transactionConfirmation.isLeaveApplied()) {
            if (transactionConfirmation.isLeaveApplied()) {
                if (transactionConfirmation.isIsleaveAppliedForFullDay()) {
                    processingRecord.setWorkedHours("00:00:00");
                } else {
                    if (transactionConfirmation.isSpecialRequestApplied()) {
                        if (transactionConfirmation.isSpecialRequestForFullDay()) {
                            if (currentDayShift.isNormalShiftWithBreakHours()) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                            } else if (breakTimeBasedOnPrecedence == null) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff)));
                            } else {

                                long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                                fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                            }
                        }
                        // handling leave and hourly special request combined
                        else if (transactionConfirmation.getIsHourlySpecialRequest()) {
                            //calculate the work hour for hourly special request
                            updateWorkHourForHrlyAndHalfDaySpecialrequest(processingRecord, workHourPolicy, currentDayShift, transactionConfirmation, breakTimeBasedOnPrecedence, specialRequestAbbrCombValidityType, fixedBreaktime, shiftDiff);
                        } else {
                            Long workHr = 0L;
                            if (processingRecord.getWhBasedOnWHPolicy() != null) {
                                workHr = DateUtil.convertHhMmSsToSeconds(processingRecord.getWhBasedOnWHPolicy());
                            }
                            if (currentDayShift.isNormalShiftWithBreakHours()) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2) + workHr));
                            } else if (breakTimeBasedOnPrecedence == null) {
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) / 2) + workHr));
                            } else {
                                long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                                if (!CollectionUtils.isEmpty(workHourPolicy.getWorkHourRuleModelList())) {
                                    for (int i = 0; i < workHourPolicy.getWorkHourRuleModelList().size(); i++) {
                                        if (workHourPolicy.getWorkHourRuleModelList().get(i).isEnableBreakType()) {
                                            List<Long> ids = workHourPolicy.getWorkHourRuleModelList().get(i).getIds();
                                            List<Long> longStream = ids.stream().filter(item -> item.equals(processingRecord.getShiftId())).collect(Collectors.toList());
                                            if (longStream != null) {
                                                fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getWorkHourRuleModelList().get(i).getBreakTime());
                                            } else {
                                                fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(breakTimeBasedOnPrecedence);
                                            }
                                        }
                                    }
                                }
                                processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeInLong) / 2) + workHr));
                            }
                        }
                    } else {
                        // leave applied for half day
                        if (processingRecord.getAttendanceStatusCode().startsWith("A") || processingRecord.getAttendanceStatusCode().endsWith("A")) {
                            processingRecord.setWorkedHours("00:00:00");
                        } else if (processingRecord.getAttendanceStatusCode().startsWith("P") || processingRecord.getAttendanceStatusCode().endsWith("P") || processingRecord.getAttendanceStatusCode().startsWith("M") || processingRecord.getAttendanceStatusCode().endsWith("M") || processingRecord.getAttendanceStatusCode().startsWith("MS") || processingRecord.getAttendanceStatusCode().endsWith("MS")) {
                            processingRecord.setWorkedHours(processingRecord.getWhBasedOnWHPolicy());
                        }
                    }
                }
            } else {
                processingRecord.setWorkedHours("00:00:00");
            }
        } else if (transactionConfirmation.isSpecialRequestApplied()) {

            if (transactionConfirmation.isSpecialRequestForFullDay()) {
                /*processingRecord.setWorkedHours(workHourPolicy.getMinThresholdWorkHourForFullday());*/

                if (specialRequestAbbreviation) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                    }
                }

            }
            //hourly request
            else if (transactionConfirmation.getIsHourlySpecialRequest()) {

                updateWorkHourForHrlyAndHalfDaySpecialrequest(processingRecord, workHourPolicy, currentDayShift, transactionConfirmation, breakTimeBasedOnPrecedence, specialRequestAbbrCombValidityType, fixedBreaktime, shiftDiff);
            } else {
                /*processingRecord.setWorkedHours(workHourPolicy.getMinThresholdWorkHourForHalfday());*/
                long workHr = 0;
                if (processingRecord.getWhBasedOnWHPolicy() != null) {
                    workHr = DateUtil.convertHhMmSsToSeconds(processingRecord.getWhBasedOnWHPolicy());
                }

                if ((isValidPresentSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("POD|PFW|PWFH|ODP|FWP|WFHP|ODWFH|WFHOD|ODFW|FWOD|FWWFH|WFHFW|MOD|MFW|MWFH|ODM|FWM|WFHM|MSOD|MSFW|MSWFH|ODMS|FWMS|WFHMS"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2) + workHr));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) / 2) + workHr)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((((shiftDiff) - fixedBreakTimeInLong) / 2) + workHr));
                    }
                } else if (isValidCombinationSpecialRequestAbbreviation) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeBtwBreakOutIn));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(shiftDiff));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        if (!CollectionUtils.isEmpty(workHourPolicy.getWorkHourRuleModelList())) {
                            for (int i = 0; i < workHourPolicy.getWorkHourRuleModelList().size(); i++) {
                                if (workHourPolicy.getWorkHourRuleModelList().get(i).isEnableBreakType()) {
                                    List<Long> ids = workHourPolicy.getWorkHourRuleModelList().get(i).getIds();
                                    List<Long> longStream = ids.stream().filter(item -> item.equals(processingRecord.getShiftId())).collect(Collectors.toList());
                                    if (longStream != null) {
                                        fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getWorkHourRuleModelList().get(i).getBreakTime());
                                    } else {
                                        fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(breakTimeBasedOnPrecedence);
                                    }
                                }
                            }
                        }
                        // long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat((shiftDiff) - fixedBreakTimeInLong));
                    }
                } else if ((isValidAbsenceSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("AOD|AFW|AWFH|ODA|FWA|WFHA"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) / 2)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeInLong) / 2));
                    }
                } else if ((isValidLOPSpecialRequestAbbreviation) || (processingRecord.getAttendanceStatusCode().matches("LOPFW|FWLOP|ALOP|LOPA|LOPOD|ODLOP|LOPWFH|WFHLOP"))) {
                    if (currentDayShift.isNormalShiftWithBreakHours()) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeBtwBreakOutIn) / 2));
                    } else if (breakTimeBasedOnPrecedence == null) {
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) / 2)));
                    } else {
                        long fixedBreakTimeInLong = DateUtil.convertHhMmSsToSeconds(workHourPolicy.getFixedBreakTime());
                        fixedBreakTimeInLong = getBreakTimeFromWorkHourRuleModel(processingRecord, workHourPolicy, fixedBreakTimeInLong);
                        processingRecord.setWorkedHours(DateUtil.convertSecondsToHhMmSsFormat(((shiftDiff) - fixedBreakTimeInLong) / 2));
                    }
                }
            }
        } else {
            processingRecord.setWorkedHours(processingRecord.getWhBasedOnWHPolicy());
        }


        // Binding value for Absence Hours
        Long requiredHoursInNumberFormat = DateUtil.convertHhMmSsToSeconds(processingRecord.getRequiredHours());
        Long workedHoursInNumberFormat = DateUtil.convertHhMmSsToSeconds(processingRecord.getWorkedHours());
        Long sum = requiredHoursInNumberFormat - workedHoursInNumberFormat;
        if (sum >= 0) {
            processingRecord.setAbsenceHours(DateUtil.convertSecondsToHhMmSsFormat(sum));
        } else {
            processingRecord.setAbsenceHours("00:00:00");
        }

        if (transactionConfirmation.isAttendanceExceptionEligibility()) {
            processingRecord.setRequiredHours(DateUtil.ZERO_HOUR_HHMMSS);
            processingRecord.setWorkedHours(DateUtil.ZERO_HOUR_HHMMSS);
            processingRecord.setAbsenceHours(DateUtil.ZERO_HOUR_HHMMSS);
        }

    }
