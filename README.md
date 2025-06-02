checkForMaxOT(fromTime,toTime,optionalOTfromTime,optionalOTtoTime){
    this.getIsWeekOfforHolidayOT();

    let calculatedNextDayTotalOt = null;

    if(!this.maxOTTime) return false;

    let totalOTRange = this.calculateTotalOTRange(fromTime, toTime, optionalOTfromTime, optionalOTtoTime);

    // if (this.daytypOfWeek == "Holiday" || this.daytypOfWeek == "WeekOff"){
    //   totalOTRange=this.deductBreakTime(totalOTRange,this.breakTime);
    // }
    let totalNextDayBreak;
    if(this.shouldDeductBreakForWoffAndHoliday){
      const totalBreakTimeWithinShift = this.calculateShiftBreakDuration();
      const totalCurrentDayBreak = this.getCurrentDayBreakWithinShift();
      totalNextDayBreak = this.calculateNextDayBreakDuration(totalCurrentDayBreak,totalBreakTimeWithinShift);
      totalOTRange = this.getRange(totalCurrentDayBreak,totalOTRange);
    }

    if(this.isSPlitRequired) {
      totalOTRange = this.getUpdatedTotalOtRange(totalOTRange);
      calculatedNextDayTotalOt =  this.getUpdatedNextDayTotalOt();
      calculatedNextDayTotalOt  = this.getRange(totalNextDayBreak, totalOTRange);
    }else if(this.totalBreakTime && this.totalBreakTime.value){
      totalOTRange =this.getRange(this.totalBreakTime.value,totalOTRange)
    }

    totalOTRange = this.roundOffGivenOtTime(totalOTRange);
    return this.validateMaxOtTime(calculatedNextDayTotalOt,totalOTRange);
  }

  private calculateNextDayBreakDuration(currentDayBreak: string, totalBreakDuration: string): string {
    if (this.isSPlitRequired) {
      const totalBreakSeconds = this.convertHhMmSsToSeconds(totalBreakDuration);
      const currentDayBreakSeconds = this.convertHhMmSsToSeconds(currentDayBreak);
      const nextDayBreakSeconds = totalBreakSeconds - currentDayBreakSeconds;
      return this.convertSecondsToHhMmSsFormat(nextDayBreakSeconds);
    }
    return "00:00:00";
  }
  private calculateShiftBreakDuration(): string {
    const breakInterval = DateUtil.calculateIntersectionTime(
      this.shiftBreakOutTime.value,
      this.shiftBreakInTime.value,
      this.fromTime.value,
      this.toTime.value
    );
    return this.calculationOfOvertime(breakInterval[0], breakInterval[1]);
  }

  private getCurrentDayBreakWithinShift(): string {

    let interSectionTimeInterval: [String, String] | null = DateUtil.calculateIntersectionTime(this.shiftBreakOutTime.value,
      this.shiftBreakInTime.value, this.fromTime.value, this.toTime.value);

    const breakStart = DateUtil.convertTimeToSeconds(interSectionTimeInterval[0].toString());
    const breakEnd = DateUtil.convertTimeToSeconds(interSectionTimeInterval[1].toString());
    const shiftStartTime = DateUtil.convertTimeToSeconds(this.shiftBreakInTime.value);

    let breakTimeInSeconds = this.calculationOfOvertime(interSectionTimeInterval[0], interSectionTimeInterval[1]);
    let totalCurrentDayBreak;

    if (interSectionTimeInterval) {
      if (this.isSPlitRequired) {
        if (breakStart > breakEnd) {
          totalCurrentDayBreak = breakEnd
        } else if (breakStart < shiftStartTime && breakEnd < shiftStartTime) {
          totalCurrentDayBreak = breakEnd - breakStart
        }
      } else {
        totalCurrentDayBreak = breakTimeInSeconds;
      }
    }
    return totalCurrentDayBreak;
  }
