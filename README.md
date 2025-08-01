private WorkHourDetails calculateNormalShiftWH(WorkHourPolicyDetails workHourPolicy, List<PunchRequest> punchList, ShiftDetails currentDayShift, LocalDateTime currentDate, ShiftProcessingContextModel shiftProcessingContextModel) {
        WorkHourDetails workHourDetails = new WorkHourDetails();
    /*
        Break Type - Fixed
     */
        Map<String, List<PunchRequest>> categorizedPunchesMap_Fixed = categorizePunches(punchList, BreakType.FIXED);
        List<PunchRequest> inPunches_Fixed = categorizedPunchesMap_Fixed.get("IN");
        List<PunchRequest> outPunches_Fixed = categorizedPunchesMap_Fixed.get("OUT");
        List<PunchRequest> inOutPunches_Fixed = categorizedPunchesMap_Fixed.get("IN_OUT");
        List<PunchRequest> pl = new ArrayList<>();
        Map<String, PunchRequest> inOUTPunches = new HashMap<>();

        if (inPunches_Fixed.size() == 1) {
            determineWorkLocation(workHourDetails, inPunches_Fixed.get(0));
        } else if (outPunches_Fixed.size() == 1) {
            determineWorkLocation(workHourDetails, outPunches_Fixed.get(0));
        } else if (inOutPunches_Fixed.size() == 1) {
            determineWorkLocation(workHourDetails, inOutPunches_Fixed.get(0));
        }


        if ((!inPunches_Fixed.isEmpty() && !outPunches_Fixed.isEmpty()) || inOutPunches_Fixed.size() >= 2) {
          // will get the first and last punch
            inOUTPunches = determineINandOUTPunch(inPunches_Fixed, outPunches_Fixed, inOutPunches_Fixed);
            PunchRequest firstPunch, lastPunch;
            firstPunch = inOUTPunches.get("FIRST_PUNCH");
            lastPunch = inOUTPunches.get("LAST_PUNCH");
            if (lastPunch == null) {
                pl.add(firstPunch);
                categorizedPunchesMap_Fixed = categorizePunches(pl, BreakType.FIXED);
                inPunches_Fixed = categorizedPunchesMap_Fixed.get("IN");
                outPunches_Fixed = categorizedPunchesMap_Fixed.get("OUT");
                inOutPunches_Fixed = categorizedPunchesMap_Fixed.get("IN_OUT");
            }
        }

        if ((inPunches_Fixed.size() > 0 && outPunches_Fixed.size() > 0) || inOutPunches_Fixed.size() >= 2) {

      /*Map<String, PunchRequest> inOUTPunches = determineINandOUTPunch(inPunches_Fixed,
          outPunches_Fixed, inOutPunches_Fixed);*/
            PunchRequest firstPunch, lastPunch;
            firstPunch = inOUTPunches.get("FIRST_PUNCH");
            lastPunch = inOUTPunches.get("LAST_PUNCH");
            if (lastPunch != null) {
                workHourDetails = calculateFixedWH(workHourPolicy, currentDayShift, currentDate, workHourDetails, firstPunch, lastPunch, shiftProcessingContextModel);
            }
        } else if (inPunches_Fixed.size() >= 1 && BreakType.FIXED.toString().equalsIgnoreCase(workHourPolicy.getNormalShiftBreakType().toString())) {
            if (inOutPunches_Fixed.size() == 1) {
                PunchRequest in = inPunches_Fixed.get(0);
                PunchRequest inOut = inOutPunches_Fixed.get(0);
                PunchRequest firstPunch, lastPunch;
                if (in.getLocalPunchTime().isBefore(inOut.getLocalPunchTime())) {
                    firstPunch = in;
                    lastPunch = inOut;
                    calculateFixedWH(workHourPolicy, currentDayShift, currentDate, workHourDetails, firstPunch, lastPunch, shiftProcessingContextModel);
                } else {
                    firstPunch = inOut;
                    setInPunchTime(currentDayShift, workHourDetails, firstPunch, currentDate, workHourPolicy, shiftProcessingContextModel);
                }
            } else {
                setInPunchTime(currentDayShift, workHourDetails, inPunches_Fixed.get(0), currentDate, workHourPolicy, shiftProcessingContextModel);
            }
        } else if (outPunches_Fixed.size() >= 1 && BreakType.FIXED.toString().equalsIgnoreCase(workHourPolicy.getNormalShiftBreakType().toString())) {
            if (inOutPunches_Fixed.size() == 1) {
                PunchRequest out = outPunches_Fixed.get(outPunches_Fixed.size() - 1);
                PunchRequest inOut = inOutPunches_Fixed.get(inOutPunches_Fixed.size() - 1);
                PunchRequest firstPunch, lastPunch;
                if (out.getLocalPunchTime().isAfter(inOut.getLocalPunchTime())) {
                    firstPunch = inOut;
                    lastPunch = out;
                    calculateFixedWH(workHourPolicy, currentDayShift, currentDate, workHourDetails, firstPunch, lastPunch, shiftProcessingContextModel);
                } else {
                    lastPunch = inOut;
                    setOutPunchTime(currentDayShift, workHourDetails, lastPunch, currentDate, workHourPolicy, shiftProcessingContextModel);
                }
            } else {
                setOutPunchTime(currentDayShift, workHourDetails, outPunches_Fixed.get(outPunches_Fixed.size() - 1), currentDate, workHourPolicy, shiftProcessingContextModel);
            }
        }
