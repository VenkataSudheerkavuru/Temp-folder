@Override
    public Boolean calculateBreakHour(WorkHourPolicyDetails workHourPolicy, ShiftDetails currentDayShift, PunchRequest firstPunch, PunchRequest lastPunch, GenericPair<Long, Long> breakTimings, Long whAsPerPolicy) {

        Long fixedBreakHourInSeconds=0L;
        Long breakHourInSeconds=0L;
        Boolean isBreakourDeducted = Boolean.TRUE;

        //case 1: if break type is fixed then calculate the break time
        if (BreakType.FIXED.equals(workHourPolicy.getNormalShiftBreakType()) || BreakType.FIXED
                .equals(workHourPolicy.getBreakShiftBreakType())) {
            breakHourInSeconds = calculateBreakHourInSecondsForFixedBreakType(workHourPolicy, currentDayShift);
            fixedBreakHourInSeconds = breakHourInSeconds;
        }

        // case 2: if break type is based on min work hours list then calculate the break time
        if(workHourPolicy.getFixedBreakTime() != null && workHourPolicy.getMinimumWorkHoursForBreak() != null &&
                workHourPolicy.getMinimumWorkHoursForBreak().getMinWorkHrsForBrk() != null &&
                workHourPolicy.getMinimumWorkHoursForBreak().getMinWorkHrsForBrk()){
            GenericPair<Boolean,Long>breakHourDetails =getBreakTimeBasedOnMinWorkHour(workHourPolicy, whAsPerPolicy, isBreakourDeducted);
           breakHourInSeconds = breakHourDetails.getSecond();
            fixedBreakHourInSeconds = breakHourInSeconds;
            isBreakourDeducted = breakHourDetails.getFirst();
        }

        //case :3 if the break type is based on workhour rule model
        if(!CollectionUtils.isEmpty(workHourPolicy.getWorkHourRuleModelList())) {
            GenericPair<Boolean,Long>breakHourDetails = getBreakHourBasedOnWorkHourRuleModelList(workHourPolicy, currentDayShift, whAsPerPolicy,isBreakourDeducted);
            breakHourInSeconds = breakHourDetails.getSecond();
            fixedBreakHourInSeconds = breakHourInSeconds;
            isBreakourDeducted = breakHourDetails.getFirst();
        }

        // in normal shift if breakIn and break out is given then priority should be given for breakIn and break out
        //case 4.a : calculate the fixed break time for normal shift with break hours
        //case 4.b : in normal shift if breakIn and break out is given then priority should be given for breakIn and break out
        if(currentDayShift.isNormalShiftWithBreakHours()  && currentDayShift.getShiftTiming()!=null &&
                currentDayShift.getShiftTiming().getBreakIn()!=null && currentDayShift.getShiftTiming().getBreakOut()!=null ) {
            //fixed breakTime
            fixedBreakHourInSeconds = getFixedBreakTimeBtwBreakInAndOut(currentDayShift.getShiftTiming(), currentDayShift.isNormalShiftWithBreakHours());

            //breaktime based on punches
           if(firstPunch!=null  && lastPunch!=null){
               long breakTimeInSeconds = calculateBreakTimeForNormalShiftWithWorkHours(currentDayShift, firstPunch, lastPunch);
               breakHourInSeconds = compareAndGetBreakTime(whAsPerPolicy, breakTimeInSeconds);
               isBreakourDeducted = true;
           }
        }

        //update the fixed breakTime and breakTime based on punches
        breakTimings.setFirst(fixedBreakHourInSeconds);
        breakTimings.setSecond(breakHourInSeconds);

        return isBreakourDeducted;

    }
