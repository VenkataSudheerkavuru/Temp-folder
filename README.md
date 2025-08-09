private BreakPriority getPriorityOfBreak(WorkHourPolicyDetails workHourPolicy, ShiftDetails currentDayShift) {

    if (currentDayShift != null && currentDayShift.isNormalShiftWithBreakHours()) {
        if (currentDayShift.getShiftTiming() != null
                && currentDayShift.getShiftTiming().getBreakIn() != null
                && currentDayShift.getShiftTiming().getBreakOut() != null) {
            return BreakPriority.NORMAL_SHIFT_WITH_BREAK_HOURS;
        }
    }

    if (workHourPolicy != null &&
            (BreakType.FIXED.equals(workHourPolicy.getNormalShiftBreakType())
                    || BreakType.FIXED.equals(workHourPolicy.getBreakShiftBreakType()))) {
        return BreakPriority.FIXED;
    }

    if (workHourPolicy != null
            && workHourPolicy.getFixedBreakTime() != null
            && workHourPolicy.getMinimumWorkHoursForBreak() != null
            && Boolean.TRUE.equals(workHourPolicy.getMinimumWorkHoursForBreak().getMinWorkHrsForBrk())) {
        return BreakPriority.MIN_WORK_HOURS;
    }

    if (workHourPolicy != null && !CollectionUtils.isEmpty(workHourPolicy.getWorkHourRuleModelList())) {
        return BreakPriority.RULE_MODEL;
    }

    return BreakPriority.NONE;
}
