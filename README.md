@Override
public void deductBreakRangeIfAnyOverlapping(
        List<GenericPair<LocalDateTime, LocalDateTime>> permissionRanges,
        ShiftDetails shiftDetails,
        LocalDateTime currentDate) {

    if (!shiftDetails.isNormalShiftWithBreakHours()) return;

    DateTimeFormatter fmt = DateTimeFormatter.ofPattern(DateUtil.TIMEFORMAT_Hmm);
    ShiftTiming shiftTiming = shiftDetails.getShiftTiming();

    // Build shift times
    LocalDateTime shiftStartDateTime = toDateTime(currentDate, shiftTiming.getStartTime(), fmt);
    LocalDateTime shiftEndDateTime   = toDateTime(currentDate, shiftTiming.getEndTime(), fmt);
    if (shiftEndDateTime.isBefore(shiftStartDateTime)) shiftEndDateTime = shiftEndDateTime.plusDays(1);

    // Build break times (within the shift)
    LocalDateTime breakStartDateTime = toDateTime(currentDate, shiftTiming.getBreakIn(), fmt);
    if (breakStartDateTime.isBefore(shiftStartDateTime)) breakStartDateTime = breakStartDateTime.plusDays(1);

    LocalDateTime breakEndDateTime = toDateTime(currentDate, shiftTiming.getBreakOut(), fmt);
    if (breakEndDateTime.isBefore(breakStartDateTime)) breakEndDateTime = breakEndDateTime.plusDays(1);

    GenericPair<LocalDateTime, LocalDateTime> breakRange = new GenericPair<>(breakStartDateTime, breakEndDateTime);

    // Remove breaks from permissions
    List<GenericPair<LocalDateTime, LocalDateTime>> updated = new ArrayList<>();
    for (GenericPair<LocalDateTime, LocalDateTime> perm : permissionRanges) {
        updated.addAll(subtractRange(perm, breakRange));
    }
    permissionRanges.clear();
    permissionRanges.addAll(updated);
}

private LocalDateTime toDateTime(LocalDateTime baseDate, String timeStr, DateTimeFormatter fmt) {
    return baseDate.toLocalDate().atTime(LocalTime.parse(timeStr, fmt));
}

private List<GenericPair<LocalDateTime, LocalDateTime>> subtractRange(
        GenericPair<LocalDateTime, LocalDateTime> original,
        GenericPair<LocalDateTime, LocalDateTime> toRemove) {

    List<GenericPair<LocalDateTime, LocalDateTime>> result = new ArrayList<>();
    GenericPair<LocalDateTime, LocalDateTime> intersection = getIntersection(original, toRemove);

    if (intersection == null) {
        result.add(original);
        return result;
    }
    if (original.getFirst().isBefore(intersection.getFirst())) {
        result.add(new GenericPair<>(original.getFirst(), intersection.getFirst()));
    }
    if (original.getSecond().isAfter(intersection.getSecond())) {
        result.add(new GenericPair<>(intersection.getSecond(), original.getSecond()));
    }
    return result;
}

private GenericPair<LocalDateTime, LocalDateTime> getIntersection(
        GenericPair<LocalDateTime, LocalDateTime> a,
        GenericPair<LocalDateTime, LocalDateTime> b) {

    LocalDateTime start = a.getFirst().isAfter(b.getFirst()) ? a.getFirst() : b.getFirst();
    LocalDateTime end = a.getSecond().isBefore(b.getSecond()) ? a.getSecond() : b.getSecond();
    return end.isBefore(start) ? null : new GenericPair<>(start, end);
}
