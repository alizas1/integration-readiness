# When a report is too long

The AI writes the whole report in one reply. When a job checks many use cases, or reads many resources, the report can pass the AI model's reply limit before it's handed in. The job then fails with no report, and you aren't charged a call for it.

## A validation that started from a review

Start 2 smaller validations, each with the failed job's ID as priorReport and 2 use cases in useCases.max. You can specify the specific use cases in useCases.ids or by title in useCases.specified to prevent overlap and make sure all are tested. Then call Combine with the review and both validations.

## A validation without a review

Start a smaller validation with the failed job's ID as priorReport and fewer use cases in useCases.max. You can specify the specific use cases in useCases.specified to make sure the ones you need are tested. Or try again with fewer resources.

## A review

Start 2 smaller reviews, each with the failed job's ID as priorReport and 2 use cases in useCases.max. You can specify the specific use cases in useCases.specified to prevent overlap and make sure all are tested. Then call Combine with both reviews. Or try again with fewer resources.

## A combined report

If more than 2 jobs went in, combine fewer jobs at a time. For now, 2 jobs that are too long together can't be combined into one report.

A full guide, with request examples, is on its way.
