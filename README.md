package in.securtime.core.processing.server.impl.WorkHourProcessing.impl;

import in.securtime.core.bmddata.api.model.PunchRequest;
import in.securtime.core.policymanagement.api.models.shifts.GraceTime;
import in.securtime.core.policymanagement.api.models.shifts.ShiftDetails;
import in.securtime.core.policymanagement.api.models.shifts.ShiftTiming;
import in.securtime.core.processing.server.impl.WorkHourProcessing.HoursCalculationService;
import in.securtime.shared.util.date.DateUtil;
import org.apache.commons.lang3.StringUtils;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.time.LocalTime;
import java.time.format.DateTimeFormatter;

@Service
public class HoursCalculationServiceImpl implements HoursCalculationService {

    @Override
    public Long[] calculateRevisedPunches(PunchRequest firstPunch, PunchRequest lastPunch, ShiftDetails currentDayShift, LocalDateTime currentDate) {

        Long firstPunchRevised;
        Long lastPunchRevised;

        Long inPunchFixed = firstPunch.getPunchTime().getTime();
        Long outPunchFixed = lastPunch.getPunchTime().getTime();

        GraceTime graceTime = currentDayShift.getGraceTime();
        ShiftTiming shiftTiming = currentDayShift.getShiftTiming();

        LocalTime shiftStart = LocalTime.parse(shiftTiming.getStartTime(), DateTimeFormatter.ofPattern(DateUtil.TIMEFORMAT_Hmm));
        LocalTime shiftInGraceTime = calculateShiftInGrace(graceTime, shiftStart);

        LocalDateTime nightShiftNextDay = currentDate;
        if (shiftInGraceTime.isBefore(shiftStart)) {
            nightShiftNextDay = currentDate.plusDays(1);
        }
        LocalDateTime shiftInGraceDateTime = nightShiftNextDay.toLocalDate().atTime(shiftInGraceTime);

        LocalTime shiftEnd = LocalTime.parse(shiftTiming.getEndTime(), DateTimeFormatter.ofPattern(DateUtil.TIMEFORMAT_Hmm));
        LocalTime shiftOutGraceTime = calculateShiftOutGrace(graceTime, shiftEnd);
        LocalDateTime shiftOutGraceDateTime = nightShiftNextDay.toLocalDate().atTime(shiftOutGraceTime);
        if (shiftOutGraceTime.isBefore(shiftEnd)) {
            shiftOutGraceDateTime = currentDate;
        }
        if (shiftOutGraceTime.isAfter(shiftEnd)) {
            shiftOutGraceDateTime = currentDate.plusDays(1);
        }

        return new Long[0];
    }

    /**
     * SIG = (SS + Shift_In_Grace_Time))
     * @param graceTime      Shift Grace Time (SGT)
     * @param shiftStartTime Shift Start Time (SS)
     * @return Shift In Grace (SIG)
     */
    private LocalTime calculateShiftInGrace(GraceTime graceTime, LocalTime shiftStartTime) {
        LocalTime shiftInGrace = shiftStartTime;

        if (StringUtils.isNotBlank(graceTime.getShiftIn())) {
            shiftInGrace = LocalTime
                    .parse(DateUtil.convertSecondsToHhMmSsFormat(Long.parseLong(graceTime.getShiftIn()) * 60),
                            DateTimeFormatter.ofPattern("H:mm:ss"));
            shiftInGrace = shiftStartTime.plusMinutes(shiftInGrace.getMinute())
                    .plusHours(shiftInGrace.getHour());
        }
        return shiftInGrace;
    }

    /**
     * SOG = (SE - Shift_Out_Grace_Time)
     *
     * @param graceTime    Shift Grace Time (SGT)
     * @param shiftEndTime Shift End Time
     * @return Shift Out Grace (SOG)
     */
    private LocalTime calculateShiftOutGrace(GraceTime graceTime, LocalTime shiftEndTime) {
        LocalTime shiftOutGrace = shiftEndTime;

        if (StringUtils.isNotBlank(graceTime.getShiftOut())) {
            shiftOutGrace = LocalTime.parse(
                    DateUtil.convertSecondsToHhMmSsFormat(Long.parseLong(graceTime.getShiftOut()) * 60),
                    DateTimeFormatter.ofPattern("H:mm:ss"));
            shiftOutGrace = shiftEndTime.minusMinutes(shiftOutGrace.getMinute())
                    .minusHours(shiftOutGrace.getHour());
        }
        return shiftOutGrace;
    }
}
