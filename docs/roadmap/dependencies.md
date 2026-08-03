# Roadmap dependencies and shared concerns

M01 is the prerequisite for all other milestones. M02, M03 and M04 form the
backbone. M05–M08 can proceed independently after M04 unless a listed contract
dependency says otherwise.

Cross-cutting requirements — security, logging, schemas, migrations and tests — live
in the relevant contract topic. Every milestone must demonstrate its end-to-end
acceptance path; a horizontal layer alone is not a completed slice.

After M02 provides the Telegram and MAX Mini App launch/session adapters, every
milestone that adds authenticated web UI verifies its main user path in the ordinary
browser and both Mini Apps unless its acceptance explicitly limits the surface.
