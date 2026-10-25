# Lessons Learned

Notes from the Inventor 2027 external iLogic rules in this repository, mainly `Grouped Mounting Hardware.iLogicVb` and `Select_Mounting_Face.iLogicVb`. Each entry is an error we hit, why it failed, and what to do in the next script.

Inventor internal length is centimeters. Show inches as the centimeter value divided by `2.54`, and write parameter expressions with an explicit unit such as `"0.125 in"` so the document unit does not change the meaning.

## How a rule is compiled

**Error.** `Reference to a non-shared member requires an object reference.` Select Mounting Face failed on the line inside `SMFFaceSelector` that called a function defined on the rule.

**Why.** iLogic compiles one external rule as one class. A class written inside the rule cannot call a function written at rule level, because that function is an instance member of the outer class. The nested class has no object to call it on.

**Next time.** Put the helper on the class that needs it, as a private method. A constant on another class is fine to read. Keep the first window of a batch rule (the folder prompt) in `Main`, before any file is opened.

**Error.** Two `Dim` statements for the same name in one `Sub`, even in different `If` blocks, fail to compile.

**Why.** In VB, a local name belongs to the whole procedure. A variable declared inside an `If` still occupies that name for the rest of the procedure.

**Next time.** Use one name for the whole procedure, or give each block its own name.

## Transactions

**Error.** `Unspecified error (0x80004005 (E_FAIL))` while placing hardware, after the holes had already succeeded.

**Why.** An iLogic rule that edits the active assembly already has an assembly transaction open. `TransactionManager.StartTransaction` on that same assembly starts a second one, and Inventor rejects it.

**Next time.** Do not start an assembly transaction inside a rule that is already editing the assembly. A transaction on the plate part document is a different document and is allowed. If grounding, a constraint, or an attribute write fails after the occurrence exists, keep the occurrence and say so. Do not roll the part back because a later assembly edit failed.

## Reading geometry

**Error.** The screw was never placed. The finish note was `Unspecified error (0x80004005 (E_FAIL))` while reading the screw.

**Why.** Threaded faces often throw `E_FAIL` when `Face.Geometry` or an edge's geometry is read. One unguarded read aborted the whole placement, before the mounting-face prompt.

**Next time.** Wrap each face and each edge in its own `Try`. Skip the face that throws. A partial model is enough to continue, and the saved mounting face replaces the guess. The same guard belongs on any walk of hole edges, washer edges, or shank cylinders.

**Error.** Inventor's own file dialog raised `E_FAIL` for some option combinations, so the user never reached the part picker.

**Why.** `FileDialog` rejects some combinations of title, filter, and initial folder.

**Next time.** Try Inventor's dialog first. On failure, open the same folder with the .NET file dialog.

**Error.** Adding a washer with `Occurrences.Add` raised `E_FAIL` even though the part had just been read.

**Why.** The rule already had the part open. A second open from inside the rule fails.

**Next time.** Add the occurrence with `AddByComponentDefinition` on the definition that is already open. Fall back to `Occurrences.Add` only if that fails, and close a hidden document the rule opened. Leave a document the user already has on screen open.

## Forms in Inventor 2027

**Error.** Setting `Form.AcceptButton` or `Form.CancelButton` crashed or pulled in a missing .NET desktop assembly.

**Why.** Inventor 2027's iLogic host has a regression around those WinForms properties.

**Next time.** Wire button `Click` handlers yourself. Do not set `AcceptButton` or `CancelButton`.

## Hole features

**Error.** A hole feature was created, the API returned success, and the hole pointed out into empty space.

**Why.** A successful `HoleFeatures` call does not mean material was removed. The through-all direction follows the sketch normal, which is independent of the face normal and the camera.

**Next time.** Before cutting, probe the solid on both sides of the sketch normal and choose the side that contains material. After one part update, check that the sample points are empty. If they are still solid, reverse that feature and update once more. Update the part once for the whole run. A separate update per group made countersinks take minutes.

**Error.** Countersink groups of two or three holes took several minutes. Plain clearance holes on the same plate were quick.

**Why.** After creating the countersink, the rule assigned `ClearanceInfo`. Inventor then re-read `Clearance.xls` on every parameter write and every document update, and the rule also rewrote the seat diameter and angle because the table had replaced them. That fight ran again for every group.

**Next time.** Use `Clearance.xls` for a plain drilled clearance hole. For a countersink, counterbore, or spotface, create the feature with the diameters and angle from the review grid and do not assign `ClearanceInfo`. Check the seat dimensions once, after the single update. The grid values are the authority. Tell the user that clearance-table previews are approximate until a plain clearance hole is created.

**Error.** `PlanarSketch.Name` raised `E_FAIL` while preparing a group, and the hole was never cut.

**Why.** Inventor rejects some names, including a collision with an existing sketch. The rename was treated as a fatal step.

**Next time.** Do not rename sketches in the preparation loop. If a visible name is still wanted, try it after the feature exists and keep going when Inventor rejects it. Store identity in attributes, not in the feature name.

**Error.** The group name was stored inside the feature identity, so renaming a group looked like a new hole specification.

**Why.** The identity string mixed the name with the size, thread, seat, and clearance data.

**Next time.** Keep each identity field in its own attribute. Inventor string attributes hold 255 characters. The specification key excludes the display name. A name change updates the existing feature. A specification change does not edit that feature, and the rule says why before it cuts.

## Existing holes

**Error.** After the plate face and the component holes were selected, the rule said the holes could not be placed because they were off the material. The plate already had holes there.

**Why.** `Face.GetClosestPointTo` on a face with a hole returns a point on the rim. The hole center is farther from that point than the on-face tolerance, so the center was classified as outside the plate. Later, the solid probe also failed, because there is no material under a hole that is already open. The boundary check then treated the existing rim as an interfering opening.

**Next time.** If the projected center misses the face and a circular edge on that face is centered on the point, the plate already has a hole. Keep the location for hardware and do not cut it. If every selected location is already open, skip the plate transaction and place hardware. A center that misses the face and has no circular rim on it is actually outside the plate. Ignore a circular edge centered on the point when checking the new hole against the face boundary.

## Subassemblies

**Error.** `Version 1 requires the plate to be a TOP-LEVEL part in this assembly (not inside a subassembly).` The plate the user clicked was a real part. It lived inside a subassembly.

**Why.** Selecting a face on a part that is nested in a subassembly returns a `FaceProxy` whose `ContainingOccurrence` is a `ComponentOccurrenceProxy`. The rule treated every proxy as unsupported and stopped before reading the part. The part document is still the `.ipt`, and a proxy obtained by picking in the open assembly reports its position in that assembly. Autodesk's assembly API states that `Transformation` on those proxies is already in the context of the assembly you picked in.

**Next time.** Edit the part from `Occurrence.Definition` whether or not the occurrence is a proxy. Keep the proxy for constraints: `CreateGeometryProxy` on it returns geometry in the assembly where the constraint is created. Use the proxy transform to move plate-space points into that assembly, and its inverse to move clicked rims back into the part. Check the transform against points on the face that was clicked. If it does not land those points, multiply each occurrence's transform inside its own parent, from the part outward. If neither transform lands the points, stop before cutting. A top-level part is not a proxy; keep using its transform directly. Leave Grounded alone while choosing the transform. Grounding the nested part would edit the subassembly.

## Selection

**Error.** Holes could only be picked one click at a time.

**Why.** `SelectEvents.WindowSelectEnabled` defaults to false. A drag box does nothing until it is turned on, and the window needs its own filter.

**Next time.** For a multi-pick of edges, set `WindowSelectEnabled` and call `AddWindowSelectionFilter` with the same filter used for clicks. Leave a single face pick as one click. A box also returns arc edges from a countersink, so keep full circles and ignore an edge that is already in the list. `OnSelect` can fire before `SelectedEntities` is refreshed, so read `JustSelectedEntities`. Shift-click still comes through `OnUnSelect`. Inventor's window keeps edges that sit fully inside the box. A box that only crosses a rim can miss that hole.

**Error.** Pressing Esc, or starting another Inventor command, left the rule waiting.

**Why.** `CommandManager.Pick` does not give a reliable cancel, and `InteractionEvents` ends on Esc through `OnTerminate`.

**Next time.** Use `InteractionEvents` plus a modeless palette with Finish and Cancel. Treat `OnTerminate` as cancel for a single selection. In a batch, treat it as skip-this-part only, and let the palette Close stop the batch.

## Hardware placement and orientation

**Error.** Both washers landed on the same side of the joint.

**Why.** The circle normal on an assembly edge proxy is not a reliable "out of the part" direction. It flipped between the two ends.

**Next time.** Capture the plate face's outward normal before any hole is cut. Place the plate-side hardware along that normal. Place the component-side hardware along the hole axis, from the clicked rim toward the outside of that part.

**Error.** The side labeled as the opposite end of the hole put the nut on the far rim of the component. The user wanted it on the plate face chosen at the start.

**Why.** The far circular edge of the component hole was treated as the other side of the joint. That edge is still on the component.

**Next time.** The two sides of the joint are the clicked component rim and the plate face selected first. The nut follows the side the user chose. The washer under the screw uses the other side. The nut stacks outside the washer under the nut. The screw head sits on the washer under the screw, or on the part when that washer is not placed.

**Error.** A nylon lock nut sat nylon-ring down.

**Why.** The rule guessed which planar face was the bearing face. On a lock nut the two ends are different, and the guess picked the ring.

**Next time.** The first time a nut or screw file is used, place a temporary copy, ask for the face, and save it on the part. For a nut, ask for the face that sits against the washer. For a nylon nut, that is the smooth metal face. Store the reference key and the key context in custom iProperties. Read them back on the next run. If either value is missing, or the key no longer binds to a face, ask again. Delete the temporary copy when the prompt ends.

**Error.** Flathead screws in countersunk holes were flipped. The head went into the hole and the tip pointed out.

**Why.** A flathead has no planar underside. The planar filter can only select the flat top. The old rule pointed the axis into the solid from the clicked face. From the top, that direction points at the shank, and placement aligns the axis with the outward direction, so the tip comes out of the hole. The same into-solid test is correct for a pan, hex, or socket head, because the clicked face is the underside and the solid is the head.

**Next time.** Find the longest cylinder that can be the shank. Point the axis away from that shank, out through the head, so the tip goes into the hole. A short cylinder smaller than the smallest circle on the clicked face is a drive recess, not the shank. If no shank is found, keep the solid probe. Re-read the saved face on every run so an already-saved flathead turns around without asking again. Tell the user to click the underside of a pan head and the flat top of a flathead.

**Error.** The browser showed names such as `MGH Nut #6 plate [13.335000,-32.067500,0.000000]`.

**Why.** The rule assigned `ComponentOccurrence.Name` after placing the part.

**Next time.** Leave the name Inventor assigns from the part file. The `:1` and `:2` suffixes are Inventor's. Still recognize the older generated names so a later run can replace those occurrences. Prefer attributes on the occurrence for identity. A name match is only the fallback for placements made before attributes existed.

## Files and iProperties

**Error.** Select Mounting Face opened Inventor backup parts and `~` temporary files, and on a path check the `OldVersions` folder was not detected.

**Why.** Inventor writes backups under a folder named `OldVersions` and temporary files whose names start with `~`. A path test that splits only on `Path.DirectorySeparatorChar` treats a Windows path with backslashes as one segment when the test runs where the separator is `/`.

**Next time.** Walk subfolders, skip a directory named `OldVersions`, and skip a file whose name starts with `~` or does not end in `.ipt`. Split paths on both `\` and `/`.

**Error.** A read-only part was changed in memory, or a part the user already had open was closed.

**Why.** The batch saved and closed every document it opened, including ones it did not open and ones Inventor would not save.

**Next time.** If the part is not modifiable, report it and leave it. If the user already had it open, save it and leave the window open. After a successful `Save` on a part this rule opened, call `Close(True)` so Inventor does not show a save prompt. Skip This Part leaves that one file unchanged. Stop ends the batch.

**Error.** A mounting face saved on a part was still used after the face had been deleted from the model.

**Why.** The iProperty string was present, but the reference key no longer bound.

**Next time.** Require both the key and the context. Bind the key before trusting it. A failed bind means the face is unset.

## Progress

**Error.** The progress bar threw, or the cancel button left the part half-edited.

**Why.** `UpdateProgress` was called more times than the number of steps passed to `CreateProgressBar`. `AllowCancel` true during an open transaction left Inventor's COM state inconsistent.

**Next time.** Count the steps before creating the bar, and tick once per real completed step. Pass `False` for allow-cancel while a transaction is open. Put the current operation name in the bar message before a slow call, and include that name in the error dialog. `DoEvents` keeps the bar painting. It also lets Inventor run other events, so do not start a second command from inside the loop.
