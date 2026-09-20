# Patterns

Worked templates. Each follows the rules in SKILL.md: flat, one command per
line, inspection at the end.

## Event-study figure, two specifications

Coefficients stored with `postfile`, drawn with `twoway`. Blue for the first
specification, orange for the second, translucent bands, series offset so the
markers do not overlap.

```stata
****************Event study, two specifications
reghdfe y DID_1 DID_2 DID_3 DID_4 DID_5 DID_7 DID_8 DID_9 treat, absorb(prov#year uni#year) cluster(major)
test DID_1 DID_2 DID_3 DID_4 DID_5

tempname p1
postfile `p1' int yr double b1 double se1 using "$data/es_s1.dta", replace
post `p1' (2017) (_b[DID_1]) (_se[DID_1])
post `p1' (2018) (_b[DID_2]) (_se[DID_2])
post `p1' (2019) (_b[DID_3]) (_se[DID_3])
post `p1' (2020) (_b[DID_4]) (_se[DID_4])
post `p1' (2021) (_b[DID_5]) (_se[DID_5])
post `p1' (2022) (0)         (0)
post `p1' (2023) (_b[DID_7]) (_se[DID_7])
post `p1' (2024) (_b[DID_8]) (_se[DID_8])
post `p1' (2025) (_b[DID_9]) (_se[DID_9])
postclose `p1'

reghdfe y DID_1 DID_2 DID_3 DID_4 DID_5 DID_7 DID_8 DID_9, absorb($FE) cluster(major)
test DID_1 DID_2 DID_3 DID_4 DID_5
test DID_4 DID_5

tempname p2
postfile `p2' int yr double b2 double se2 using "$data/es_s2.dta", replace
post `p2' (2017) (_b[DID_1]) (_se[DID_1])
post `p2' (2018) (_b[DID_2]) (_se[DID_2])
post `p2' (2019) (_b[DID_3]) (_se[DID_3])
post `p2' (2020) (_b[DID_4]) (_se[DID_4])
post `p2' (2021) (_b[DID_5]) (_se[DID_5])
post `p2' (2022) (0)         (0)
post `p2' (2023) (_b[DID_7]) (_se[DID_7])
post `p2' (2024) (_b[DID_8]) (_se[DID_8])
post `p2' (2025) (_b[DID_9]) (_se[DID_9])
postclose `p2'

use "$data/es_s1.dta", clear
merge 1:1 yr using "$data/es_s2.dta", nogen

gen lo1 = b1 - 1.96*se1
gen hi1 = b1 + 1.96*se1
gen lo2 = b2 - 1.96*se2
gen hi2 = b2 + 1.96*se2
gen yr1 = yr - 0.06
gen yr2 = yr + 0.06

list yr b1 b2

twoway (rarea lo1 hi1 yr1, color("$BLUE%12") lwidth(none)) ///
       (rarea lo2 hi2 yr2, color("$ORANGE%12") lwidth(none)) ///
       (connected b1 yr1, lcolor("$BLUE") lwidth(medthick) msymbol(O) msize(medium) mcolor("$BLUE")) ///
       (connected b2 yr2, lcolor("$ORANGE") lwidth(medthick) msymbol(O) msize(medium) mcolor("$ORANGE")), ///
	yline(0, lpattern(dot) lcolor(gs6)) ///
	xline(2022.5, lpattern(dash) lcolor(gs8)) ///
	xlabel(2017(1)2025, labsize(small)) ///
	ylabel(, labsize(small) angle(0) grid glcolor(gs14) glwidth(vthin)) ///
	ytitle("Change in outcome", size(small)) ///
	xtitle("") ///
	legend(order(3 "Specification one" 4 "Specification two") ///
	       position(6) rows(1) region(lstyle(none)) size(small)) ///
	graphregion(color(white)) plotregion(color(white) margin(medium)) ///
	ysize(4) xsize(6.5)

graph export "$figs/fig_es.pdf", replace
graph export "$figs/fig_es.png", replace width(2400)
```

With a single specification, drop the second block, the offset and the legend,
and use orange.

## Binned-coefficient figure

Categories on the x axis, so points and bars with no connecting line.

```stata
****************Post-period change by score bin
reghdfe y ib7.bin##i.post, absorb(prov#year uni#year) cluster(major)

tempname pf
postfile `pf' int bin double b double se using "$data/bins.dta", replace
post `pf' (3) (_b[3.bin#1.post]) (_se[3.bin#1.post])
post `pf' (4) (_b[4.bin#1.post]) (_se[4.bin#1.post])
post `pf' (5) (_b[5.bin#1.post]) (_se[5.bin#1.post])
post `pf' (6) (_b[6.bin#1.post]) (_se[6.bin#1.post])
post `pf' (7) (0)                (0)
post `pf' (8) (_b[8.bin#1.post]) (_se[8.bin#1.post])
post `pf' (9) (_b[9.bin#1.post]) (_se[9.bin#1.post])
postclose `pf'

use "$data/bins.dta", clear
gen lo = b - 1.96*se
gen hi = b + 1.96*se
list

twoway (rbar lo hi bin, barwidth(0.06) color("$ORANGE%30")) ///
       (scatter b bin, msymbol(O) msize(medlarge) mcolor("$ORANGE")), ///
	yline(0, lpattern(dot) lcolor(gs6)) ///
	xlabel(3 "2-3" 4 "3-4" 5 "4-5" 6 "5-6" 7 "6-7" 8 "7-8" 9 "8-9", labsize(small) noticks) ///
	xscale(range(2.6 9.4)) ///
	ylabel(, labsize(small) angle(0) grid glcolor(gs14) glwidth(vthin)) ///
	ytitle("Post-period change in outcome", size(small)) ///
	xtitle("Score", size(small)) ///
	legend(off) ///
	graphregion(color(white)) plotregion(color(white) margin(medium)) ///
	ysize(4) xsize(6.5)

graph export "$figs/fig_bins.pdf", replace
graph export "$figs/fig_bins.png", replace width(2400)
```

## Concentration index (HHI)

Industry HHI of each fund's portfolio, one row per fund.

```stata
****************Industry concentration of each fund's portfolio
use "PC基本信息.dta", clear
keep 统一社会信用代码PC 国标行业门类
merge 1:m 统一社会信用代码PC using "对外投资PC_name1.dta"
tab _merge
keep if _merge == 3
keep fundID 国标行业门类

count if 国标行业门类 == ""

bysort fundID: gen n = _N
bysort fundID 国标行业门类: gen x = _N
bysort fundID 国标行业门类: keep if _n == 1
gen p = x / n
bysort fundID: egen hhi_ind = total(p^2)
bysort fundID: gen n_ind = _N
bysort fundID: keep if _n == 1
keep fundID n_ind hhi_ind

sum hhi_ind n_ind
```

If the `count` is not zero, those investments are counted as one extra industry,
which raises `n_ind` by one and adds their share to the index. Whether to drop
them first is the user's decision, so say it in prose rather than dropping them in code.
