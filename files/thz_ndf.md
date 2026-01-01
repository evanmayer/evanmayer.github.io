# 12/31/2025 [TIM] NDF Process Notes

## General Description

1. Cut 1/16" HDPE sheet into shape of NDF. Must fit in vacuum chamber and cover as much of the window as possible.
2. Clean HDPE sheet (still figuring this out):
	1. 70% iso rinse
	2. Single-pass cotton pad sweep (kimwipes?)
	3. 99% iso rinse and dry
	4. Air blow dry to remove any remaining fuzzies
3. Put HDPE sheet in bending jig
	1. Bending jig curves sheet into a section of a cylinder, so that it lies at a constant radial distance from the filament, for more uniform coverage - radius depends on chamber volume, 3.5"
4. Prepare 99.99% pure aluminum wire on filament
	1. Calculate mass of wire needed to achieve desired coating thickness, given the radius of the sheet from the filament
	2. Distribute as evenly as possible over filament
	3. Crimp onto filament using tweezers so vibrations don't shake it off under vacuum
5. Pump down to operating pressure
	1. Depends on path distance between filament and surface - >2-3x MFP lengths desired
6. Turn on power supply
	1. Note total time between power switch on and off, and total time between Al wetting to filament and power off. Power on time for consistency, and time after wetting as the independent variable as a proxy for deposition thickness
7. Turn off power suppy
8. Turn off turbo
9. Let turbo spin down until inaudible
10. Slowly bleed air into chamber and unload
11. Immediately put filter in ziploc freezer bag for mild protection from scratches
12. Record properties on sticker for bag

## Table of NDF Process Properties

| ID    | Date of Deposition | Al Charge (g) | Filament Time After Wetting (s) | Total Filament Time (s) | Filament Model Number                 | Chamber Pressure (Torr) | Distance from Filament (inch) | Bend Radius (inch) | Cleaning Notes                                                   | Chamber Notes                                        | Visual Notes                                                                       |
| ----- | ------------------ | ------------- | ------------------------------- | ----------------------- | ------------------------------------- | ----------------------- | ----------------------------- | ------------------ | ---------------------------------------------------------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------- |
| NDF00 | 2025-12-30         | 0.0125        | 5                               | 20                      | McMaster 3408N31, RDMathis F4-3X.025W | 2.7e-4                  | 3.5                           | 3.5                | Dish soap wash, CaCO3 slurry scrub, Tap water rinse, 99% iso dry | Total charge suspended from center of filament       | Visual backlit nonuniformity - center dark band                                    |
| NDF01 | 2025-12-31         | 0.0125        | 5                               | 20                      | McMaster 3408N31, RDMathis F4-3X.025W | 2.4e-4                  | 3.5                           | 3.5                | 99% iso rinse, cotton pad scrub, 99% iso rinse                   | Total charge divided into 2 canes spaced ~1.5" apart | Visual backlit nonuniformity - off-center dark band                                |
| NDF02 | 2025-12-31         | 0.0125        | 10                              | 30                      | McMaster 3408N31, RDMathis F4-3X.025W | 2.4e-4                  | 3.5                           | 3.5                | 70% iso rinse, cotton pad swipe, 70% iso rinse                   | Total charge divided into 2 canes spaced ~1.5" apart | Visual backlit nonuniformity - slow variation across narrow dim, thinnest at edges |