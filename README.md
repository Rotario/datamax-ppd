# Datamax E-4305A DPL driver for CUPS

Prints PDFs (4x6in shipping labels) on the Datamax E-4305A at `socket://192.168.50.31`
using native DPL, instead of the printer's ZPL emulation via the Zebra driver.

- `pdftodpl` — CUPS filter. Rasterises each page at 300dpi, downloads it to printer
  DRAM as a 1-bit BMP, prints it, deletes it.
- `datamax-dpl.ppd` — PPD that routes PDFs through `pdftodpl`; 4x6in only.

## Install

```
sudo install -m 755 -o root -g root pdftodpl /usr/lib/cups/filter/pdftodpl
sudo lpadmin -p Datamax_direct_ethernet -P datamax-dpl.ppd -o PageSize=w288h432
```

Then print as usual: `lp -d Datamax_direct_ethernet -P 2 label.pdf`

## Why

Under ZPL the printer kept faulting partway down a 6in label: its maximum label length
kept resetting to 5in, so it gave up looking for the next gap and raised paper-out.
Every DPL job now starts with `<STX>n` (inch units) and `<STX>M0650` (6.5in max length).

The `<STX>n` matters: the printer is configured metric (`UMM` in `<STX>KcQQQ`), where
`M0650` means 65mm (~2.55in) and every label faults.

The printer ignores a BMP's palette and prints 0 bits as black, so the filter inverts
the PBM bits (PBM's 1 = black) before sending.

## Debugging

```
printf '\x01A'       | nc -q 3 192.168.50.31 9100   # status; 2nd char Y = paper out/fault
printf '\x02KcQQQ\r' | nc -q 4 192.168.50.31 9100   # config dump (UM units, ML max length)
./pdftodpl 1 me t 1 "" label.pdf > out.dpl && nc -q 3 192.168.50.31 9100 < out.dpl
```
