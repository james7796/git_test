/* USRLIB MODULE INFORMATION

	MODULE NAME: KHE_BVDSS1
	MODULE RETURN TYPE: double 
	NUMBER OF PARMS: 13
	ARGUMENTS:
		d,	int,	Input,	,	,	
		g,	int,	Input,	,	,	
		s,	int,	Input,	,	,	
		sub,	int,	Input,	,	,	
		vdsstart,	double,	Input,	,	,	
		vdsstop,	double,	Input,	,	,	
		nstep,	int,	Input,	,	,	
		ipgm,	double,	Input,	,	,	
		udelay,	double,	Input,	10e-3,	,	
		type,	char,	Input,	,	,	
		nplc,	double,	Input,	1.0,	,	
		bvdssVal,	D_ARRAY_T,	Output,	,	,	
		bvdssValSize,	int,	Input,	,	,	
	INCLUDES:
#include <stdio.h>
#include <stdlib.h>
#include <lptdef.h>
#include <lptdef_lowercase.h>
#include <math.h>
	END USRLIB MODULE INFORMATION
*/
/* USRLIB MODULE HELP DESCRIPTION
Please contact me as soon as you can
: james@khelec.com (H.P:010-4409-4245)
	END USRLIB MODULE HELP DESCRIPTION */
/* USRLIB MODULE VERSION CONTROL */
static char const vcid[] ="$Id: KHE_BVDSS1.c Local $";
/* USRLIB MODULE PARAMETER LIST */
#include <stdio.h>
#include <stdlib.h>
#include <lptdef.h>
#include <lptdef_lowercase.h>
#include <math.h>

double KHE_BVDSS1( int d, int g, int s, int sub, double vdsstart, double vdsstop, int nstep, double ipgm, double udelay, char type, double nplc, double *bvdssVal, int bvdssValSize )
{
/* USRLIB MODULE CODE */
    double    i_range;    /* 125% of ipgm for limiti/rangei */
    double    aipgm;        /* absolute value of current */
    double    avlow;        /* absolute value of vcemin */
    double    avhigh;        /* absloute value of vcemax */
    double    sdelay;        /* sweep delay = internal + udelay */
    double    vsign;        /* voltage sign, based on N or P type */
    double    bvdss;    /* temporary storage variable */
    double    *forceArray;
    double    *MeasureArray;
    int ret;
    int k;

/* determine signs for swept voltage based upon device type */

    if ((type == 'N') || (type == 'n'))
        vsign = 1.0;
    else if ((type == 'P') || (type == 'p'))
        vsign = -1.0;
    else
        return(-1.0);        /* return with error */

/* take absolute value of current trigger and voltage start/stop */

    aipgm = fabs(ipgm);
    avlow = fabs(vdsstart);
    avhigh = fabs(vdsstop);

/* calculate total delay necessary for bsweepv call */

    //sdelay = tdelay(1, aipgm, avhigh/nstep) + udelay;
   sdelay = udelay;

/*Init */
   forceArray = (double*)malloc(sizeof(double)*nstep);
   MeasureArray = (double*)malloc(sizeof(double)*nstep);
/* connect device */

    conpin(SMU1, d, KI_EOC);

    if (sub > 0)
        conpin(SMU1L, GND, s, g, sub, KI_EOC);
    else
        conpin(SMU1L, GND, s, g, KI_EOC);


/* set current limit and range for optimal performance */

    i_range = 1.25*aipgm;            /* 25% headroom */
    setmode (SMU1, KI_INTGPLC, nplc);
    setmode (SMU1, KI_LIM_MODE, KI_VALUE);  /* Set the smu to return the actual value at limit rather than 7.e22) */
    limiti(SMU1, i_range);
    rangei(SMU1, i_range);

/* determine sense of current trigger based upon device type */

    if ((type == 'N') || (type == 'n'))
        trigig(SMU1, aipgm);
    else
        trigil(SMU1, -aipgm);

/* sweep voltage, return breakdown voltage */

    rtfary(forceArray);
    rttrigary(MeasureArray);

    bsweepv(SMU1, vsign*avlow, vsign*avhigh, nstep, sdelay, &bvdss);  
    
    for(k=0;k<nstep;k++)
    { 
      bvdssVal[k] = MeasureArray[k]; //for plot    
      printf("forceArray:%g,Measure:%g\n",forceArray[k],bvdssVal[k]);      
    }

    free(forceArray);
    free(MeasureArray);

    devint();

/* check voltage limits, return error in within 98% of low/high */
    

    if (fabs(vsign*avlow - bvdss) < 1.E-3)
        bvdss = 1.E+21;
    if (fabs(vsign*avhigh - bvdss) < 1.E-3)
        bvdss = 2.E+21;

    return(bvdss);
/* USRLIB MODULE END  */
} 		/* End KHE_BVDSS1.c */

