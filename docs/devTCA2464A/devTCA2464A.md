Source code test (custom ExtMon EPICS Driver)
=============================================

Here is a test of rendering source code (C) of an EPICS driver

```c
/* ++

  devTCA2464a.c
  m jones - Aug 29, 2017

-- */

static char what[] =
"@(#)devTCA2464A v0.1 support for TCA2464A bus expander, mjones, Aug 2017";

#define DEBUG_ON

// #define TCA2464A_ADDR 0x22

#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <string.h>
#include <linux/i2c.h>
#include <linux/i2c-dev.h>
#include <sys/ioctl.h>
#include <sys/types.h>
#include <fcntl.h>

#include "alarm.h"
#include "cvtTable.h"
#include "dbDefs.h"
#include "dbAccess.h"
#include "recGbl.h"
#include "recSup.h"
#include "devSup.h"
#include "biRecord.h"
#include "boRecord.h"
#include "link.h"
#include "epicsExport.h"

static long init_tca2464a_bi_record();
static long init_tca2464a_bi();
static long init_tca2464a_bo_record();
static long init_tca2464a_bo();
static long read_tca2464a();
static long write_tca2464a();

struct s_tca2464a_access {
  int fd;
  int addr;
  int reg;
  int bit;
};
```
We can even break it up a little bit!

``` c
struct {
  long number;
  DEVSUPFUN report;
  DEVSUPFUN init;
  DEVSUPFUN init_record;
  DEVSUPFUN get_ioint_info;
  DEVSUPFUN read_bi;
  DEVSUPFUN special_linconv;
}
devBiTCA2464A = { 6, NULL, init_tca2464a_bi, init_tca2464a_bi_record, NULL, read_tca2464a, NULL };

struct {
  long number;
  DEVSUPFUN report;
  DEVSUPFUN init;
  DEVSUPFUN init_record;
  DEVSUPFUN get_ioinit_info;
  DEVSUPFUN write_bo;
  DEVSUPFUN special_linconv;
}
devBoTCA2464A = { 6, NULL, init_tca2464a_bo, init_tca2464a_bo_record, NULL, write_tca2464a, NULL };
  
epicsExportAddress(dset,devBiTCA2464A);
epicsExportAddress(dset,devBoTCA2464A);

static int fd[2] = { -1, -1 };

unsigned char i2c_read_byte(int fd,unsigned char addr,unsigned char reg) {
  struct i2c_rdwr_ioctl_data rbd;
  struct i2c_msg messages[2];
  unsigned char buf;

  rbd.nmsgs = 2;
  rbd.msgs = messages;

  messages[0].addr = addr;
  messages[0].flags = 0;
  messages[0].len = 1;
  messages[0].buf = &reg;

  messages[1].addr = addr;
  messages[1].flags = I2C_M_RD;
  messages[1].len = 1;
  messages[1].buf = &buf;

  ioctl(fd,I2C_RDWR,&rbd);

  return buf;
}

unsigned short i2c_read_word(int fd,unsigned char addr,unsigned char reg) {
  struct i2c_rdwr_ioctl_data rbd;
  struct i2c_msg messages[2];
  unsigned char buf[2];

  rbd.nmsgs = 2;
  rbd.msgs = messages;

  messages[0].addr = addr;
  messages[0].flags = 0;
  messages[0].len = 1;
  messages[0].buf = &reg;

  messages[1].addr = addr;
  messages[1].flags = I2C_M_RD;
  messages[1].len = 2;
  messages[1].buf = buf;

  ioctl(fd,I2C_RDWR,&rbd);

  return (buf[0]<<8)|buf[1];
}

int i2c_read_buf(int fd,unsigned char addr,unsigned char *buf,int len) {
  struct i2c_rdwr_ioctl_data rbd;
  struct i2c_msg message;
  int ierr;

  rbd.nmsgs = 1;
  rbd.msgs = &message;

  message.addr = addr;
  message.flags = I2C_M_RD;
  message.len = len;
  message.buf = buf;

  ierr = ioctl(fd,I2C_RDWR,&rbd);
  return ierr;
}

int i2c_write_cmd(int fd,unsigned char addr,unsigned char cmd) {
  struct i2c_rdwr_ioctl_data rbd;
  struct i2c_msg message;

  rbd.nmsgs = 1;
  rbd.msgs = &message;

  message.addr = addr;
  message.flags = 0;
  message.len = 1;
  message.buf = &cmd;

  return ioctl(fd,I2C_RDWR,&rbd);
}

int i2c_write_byte(int fd,unsigned char addr,unsigned char reg,unsigned char val) {
  struct i2c_rdwr_ioctl_data rbd;
  struct i2c_msg message;
  unsigned char buf[2];

  rbd.nmsgs = 1;
  rbd.msgs = &message;

  message.addr = addr;
  message.flags = 0;
  message.len = 2;
  message.buf = buf;
  buf[0] = reg;
  buf[1] = val;

  return ioctl(fd,I2C_RDWR,&rbd);
}

static long init_tca2464a_bi(int after) {
#ifdef DEBUG_ON 
  printf("devTCA2464A (init) called, pass=%d\n", after);
#endif
  if ( fd[0] < 0 ) {
    fd[0] = open("/dev/i2c-1",O_RDWR);
  }
  if ( fd[1] < 0 ) {
    fd[1] = open("/dev/i2c-2",O_RDWR);
  }
  return(0);
}

static long init_tca2464a_bi_record(struct biRecord *pbi) {
  struct s_tca2464a_access *pvt;
  const char *p;
  char *q;
  int i;
#ifdef DEBUG_ON
  printf("devTCA2464A:init_tca2464a_bi_record(%p) called\n",pbi);
  printf("  pbi->inp.text = %s\n", pbi->inp.text );
#endif
  pvt = (struct s_tca2464a_access *)malloc(sizeof(struct s_tca2464a_access));
  pbi->dpvt = pvt;
  pvt->fd = -1;
  pvt->addr = -1;
  pvt->reg = -1;
  pvt->bit = -1;

  p = pbi->inp.text;
  i = strtol(p,&q,0);
  if ( i >= 0 && i <= 1 ) pvt->fd = fd[i];
  p = q;
  if ( p != NULL ) {
    pvt->addr = strtol(p,&q,0);
    p = q;
  }
  if ( p != NULL ) {
    pvt->reg = strtol(p,&q,0);
    p = q;
  }
  if ( p != NULL ) {
    pvt->bit = strtol(p,&q,0);
    p = q;
  }
  
  pbi->udf = FALSE;
  return(0);
}

static long init_tca2464a_bo(int after) {
#ifdef DEBUG_ON 
  printf("devTCA2464A:init_tca2464a_bo() called, pass=%d\n", after);
#endif
  if ( fd[0] < 0 ) {
    fd[0] = open("/dev/i2c-1",O_RDWR);
  }
  if ( fd[1] < 0 ) {
    fd[1] = open("/dev/i2c-2",O_RDWR);
  }
  return(0);
}

static long init_tca2464a_bo_record(struct boRecord *pbo) {
  struct s_tca2464a_access *pvt;
  const char *p;
  char *q;
  int i;

#ifdef DEBUG_ON
  printf("devTCA2464A:init_tca2464a_bo_record(%p) called\n",pbo);
  printf("  pbo->out.text = %s\n", pbo->out.text );
#endif
  pvt = (struct s_tca2464a_access *)malloc(sizeof(struct s_tca2464a_access));
  pbo->dpvt = pvt;
  pvt->fd = -1;
  pvt->addr = -1;

  p = pbo->out.text;
  i = strtol(p,&q,0);
  if ( i >= 0 && i <= 1 ) pvt->fd = fd[i];
  p = q;
  if ( p != NULL ) {
    pvt->addr = strtol(p,&q,0);
    p = q;
  }
  if ( p != NULL ) {
    pvt->reg = strtol(p,&q,0);
    p = q;
  }
  if ( p != NULL ) {
    pvt->bit = strtol(p,&q,0);
    p = q;
  }
  
  pbo->udf = FALSE;
  return(0);
}

static long read_tca2464a(struct biRecord *pbi) {
  const struct s_tca2464a_access *pvt;
  unsigned char buf;
  
#ifdef DEBUG_ON 
  printf("devTCA2464A:read_tca2464a(%p) accessed\n",pbi);
#endif
  pbi->udf = FALSE;

  pvt = (struct s_tca2464a_access *)pbi->dpvt;
  pbi->rval = 0;
  buf = i2c_read_byte(pvt->fd,pvt->addr,pvt->reg);
  if ( (buf&(1<<pvt->bit)) ) pbi->rval = 1;

#ifdef DEBUG_ON 
  printf("devTCA2464A:read_tca2464a(%p) 0x%02x.%d = 0x%02x --> %d\n", pbi, pvt->reg, pvt->bit, buf, pbi->rval);
#endif
  return(0);
}

static long write_tca2464a(struct boRecord *pbo) {
  const struct s_tca2464a_access *pvt;
  unsigned char buf;
  
#ifdef DEBUG_ON 
  printf("devTCA2464A:write_tca2464a(%p) accessed\n",pbo);
  printf("   oval = %d\n", pbo->rval );
#endif
  pbo->udf = FALSE;
  pvt = (struct s_tca2464a_access *)pbo->dpvt;
  buf = i2c_read_byte(pvt->fd,pvt->addr,pvt->reg);
#ifdef DEBUG_ON
  printf("  reg 0x%02x = 0x%02x --> ", pvt->reg, buf );
#endif
  buf &= ~(1<<pvt->bit);
  if ( pbo->rval ) buf |= (1<<pvt->bit);
#ifdef DEBUG_ON
  printf("0x%02x\n", buf );
#endif
  i2c_write_byte(pvt->fd,pvt->addr,pvt->reg,buf);

  return(0);
}
```

