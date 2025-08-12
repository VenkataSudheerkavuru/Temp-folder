private List<GenericPair<LocalDateTime, LocalDateTime>> mergeOverlappingRanges(
        List<GenericPair<LocalDateTime, LocalDateTime>> combinedRanges) {

    if (combinedRanges == null || combinedRanges.isEmpty()) {
        return Collections.emptyList();
    }

    // Sort by start time
    combinedRanges.sort(Comparator.comparing(GenericPair::getFirst));

    List<GenericPair<LocalDateTime, LocalDateTime>> mergedRanges = new ArrayList<>();

    // Start with the first interval
    LocalDateTime currentStart = combinedRanges.get(0).getFirst();
    LocalDateTime currentEnd = combinedRanges.get(0).getSecond();

    for (int i = 1; i < combinedRanges.size(); i++) {
        LocalDateTime nextStart = combinedRanges.get(i).getFirst();
        LocalDateTime nextEnd = combinedRanges.get(i).getSecond();

        if (!nextStart.isAfter(currentEnd)) {
            // Overlaps → extend the current end if needed
            currentEnd = currentEnd.isAfter(nextEnd) ? currentEnd : nextEnd;
        } else {
            // No overlap → save current range and move to next
            mergedRanges.add(new GenericPair<>(currentStart, currentEnd));
            currentStart = nextStart;
            currentEnd = nextEnd;
        }
    }

    // Add the last range
    mergedRanges.add(new GenericPair<>(currentStart, currentEnd));

    return mergedRanges;
}
