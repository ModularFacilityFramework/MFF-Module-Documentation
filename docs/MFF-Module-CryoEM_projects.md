# CryoEM module: projects

Module `cryoem` (v0.1.40 and later; needs MFF-core v0.1.227 or later for the project tabs). Samples and BAG
time-request proposals belong to a Science project. `module.manifest.json` has `"requires": ["science"]`:
install Science first (core refuses the install or update with 409 otherwise).

## The page
- Core's project tabs (`coreApi.ui.ProjectTabs`, remembered as `cryoem.dashboard`) are at the top of the
  CryoEM page. The Sample and Proposal pages list only the selected project's records. The Project template tab
  lists the records made before projects.
- A new record goes into the selected project. In a tab where the user cannot create (template, read-only,
  archived, or no project of their own) the new-record button is greyed out with core's reason.
- Each project tab keeps its own open record and unsaved draft, across leaving and reloading. A draft saved
  before the tabs appears on the tab it belongs to (an unsaved new record: on the first tab opened where it can
  be saved).
- To move a saved record: edit it, "Move to another project…", pick the project (core's `ProjectPicker`), save.
- On a core older than v0.1.227 a note says so, every readable record is listed and nothing new can be created.

## Who may do what
Access comes from the record's project (`coreApi.projects.access`). Core's module access to CryoEM is still
needed first.

| Access to the project | Sample or proposal in it |
|---|---|
| none | Not listed. Every route on it answers 404. |
| read | Listed, greyed, with the reason. No change, delete or picture upload (403). A proposal can be viewed and exported. |
| write or higher | Create, change, delete (proposals), add and remove pictures. |
| any, project archived | As read, with "Project ... is archived." |

- Creating a record needs write access to the selected project. Moving a record needs write access to the old
  and the new project. The server checks this on every call; the greyed tabs and forms are only a screen aid.
- Having made a record gives no extra rights: it follows the project.
- Samples have no delete function. A sample the user cannot change has no read-only form: its list row is all
  the page shows (a proposal in a readable project can offer that sample's details under "Nature of sample").
- Without a working Science module, records in a project are not shown and nothing new can be saved.

## Records made before projects
- The first time a list is opened after the update, records with no project are put into Science's "Project
  template" project. Nothing has to be run by hand. While Science cannot take them they stay as they were, and
  the next list tries again.
- Being in the template changes nobody's access to them in CryoEM. An older sample stays visible to and
  editable by every user of the module. An older proposal stays visible to and editable by its owner only.
- In Science, each user sees only their own items in the template (the template's owner, administrators and
  developers see all). An older proposal is listed there as "CryoEM proposal <id>", without its title.
- An older record can be saved without moving it. Moving it to a project makes it follow that project.
  Nothing new can be put into the template, not even by someone with write access to it.

## Project items
- Each sample and proposal is listed on its project's page in Science (item types `sample` and `proposal`;
  title: the sample acronym or the proposal title; the link opens the CryoEM tab).
- Moving or deleting a record moves or removes its item.
- If Science cannot be updated, the record is still saved and the page shows a warning. Saving again retries.
  A delete whose item removal fails also shows a warning.

## Tables
`cryoem_samples` and `cryoem_proposals` have the nullable columns `project_id`, `in_template`,
`item_project_id` and `item_in_template`. An existing installation needs core's Update (apply schema).

## API answers
Sample and proposal lists and saves add `project_id`, `in_template`, `project_label`, `can_write`,
`read_only_reason`; saves can add `project_item_warning`. In a save's answer, `item_project_id` and
`item_in_template` show the values from before the item was updated.
