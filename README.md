private List<GenericPair<LocalDateTime, LocalDateTime>> mergeOverlappingRanges(List<GenericPair<LocalDateTime, LocalDateTime>> combinedRanges) {
        List<GenericPair<LocalDateTime, LocalDateTime>> mergedRanges = new ArrayList<>();

        if (combinedRanges.size() > 1) {
            LocalDateTime start = null;
            LocalDateTime end = null;

            for (GenericPair<LocalDateTime, LocalDateTime> range : combinedRanges) {
                LocalDateTime currentStart = range.getFirst();
                LocalDateTime currentEnd = range.getSecond();

                if (start == null || currentStart.isBefore(start)) {
                    start = currentStart;
                }

                if (end == null || currentEnd.isAfter(end)) {
                    end = currentEnd;
                }
            }

            mergedRanges.add(new GenericPair<>(start, end));
        } else {
            mergedRanges.addAll(combinedRanges);
        }

        return mergedRanges;
    }
