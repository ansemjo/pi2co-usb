
project := pi2co-usb
fab := jlcpcb

.PHONY: panel
panel: panel.kicad_pcb
panel.kicad_pcb: $(project).kicad_pcb panelize.json
	cp -f $(project).kicad_pro panel.kicad_pro
	kikit panelize -p panelize.json $< $@
	rm panel.kicad_pro

.PHONY: gerber
gerber: $(project).zip
$(project).zip: panel.kicad_pcb
	kikit fab $(fab) --no-drc $< ./
	mv gerbers.zip $@
