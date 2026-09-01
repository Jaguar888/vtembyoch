# External data disclosure

Every external dataset used for training, pre-training, or fine-tuning must be listed here **before
it is ingested**. Disclosure is what makes compliance checkable — an undisclosed source is a rules
violation whether or not it changed the result.

## The Task 1 exclusion window

Permitted: stages **at or before E9.5**, and stages **strictly after E13.5**.

Excluded: everything from after E9.5 **up to and including E13.5**. E10.5 and E12.5 absolutely.
E13.5 itself is excluded — it is the far edge of the protected window, not the first point outside
it. E9.5 exactly is permitted, because it is a released training stage.

The stage label is not the test. A dataset labelled by somite count or Theiler stage that lands in
the window is excluded on the same terms, as is any model pre-trained on such data. When a source
mixes permitted and excluded stages, the filter must run at load time as a hard assertion — not as
a notebook step someone remembers to re-run.

## Log

| Source | Accession / URL | Stages included | Stages used after filter | Licence | Where it enters the pipeline | Reviewed |
|---|---|---|---|---|---|---|
| _(none yet)_ | | | | | | |

## Review notes

Record here any source that needed a judgement call, and what the call was.
