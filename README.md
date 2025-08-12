@Override
    public void deductBreakRangeIfAnyOverlapping(List<GenericPair<LocalDateTime, LocalDateTime>> permissionRanges, ShiftDetails shiftDetails, LocalDateTime currentDate) {
        if(shiftDetails.isNormalShiftWithBreakHours()){

            ShiftTiming shiftTiming = shiftDetails.getShiftTiming();

            // 1.Extract break timing
            LocalTime breakInLocalTime = LocalTime.parse(shiftDetails.getShiftTiming().getBreakIn(), DateTimeFormatter.ofPattern(DateUtil.TIMEFORMAT_Hmm));
            LocalTime breakOutLocalTime = LocalTime.parse(shiftDetails.getShiftTiming().getBreakOut(), DateTimeFormatter.ofPattern(DateUtil.TIMEFORMAT_Hmm));

            // 2. Parse shift times

            LocalTime shiftStart = LocalTime.parse(shiftTiming.getStartTime(), DateTimeFormatter.ofPattern(DateUtil.TIMEFORMAT_Hmm));
            LocalTime shiftEnd = LocalTime.parse(shiftTiming.getEndTime(), DateTimeFormatter.ofPattern(DateUtil.TIMEFORMAT_Hmm));
            LocalDateTime shiftStartDateTime = currentDate.toLocalDate().atTime(shiftStart);
            LocalDateTime shiftEndDateTime = currentDate.toLocalDate().atTime(shiftEnd);

            if (shiftEndDateTime.isBefore(shiftStartDateTime)) {
                shiftEndDateTime = shiftEndDateTime.plusDays(1);
            }
            
            

        }
    }
