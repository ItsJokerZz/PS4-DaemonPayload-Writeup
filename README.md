# How to Run a Daemon on the PS4 via Payload and SPRX

### Prerequisites:
- [PS4 Payload SDK](https://github.com/stooged/ps4-payload-sdk)

- [OpenOrbis Toolchain](https://github.com/OpenOrbis/OpenOrbis-PS4-Toolchain)

Download the required SDKs and setup the Environment(s) for each if you haven't already.
<br><br>

## Example of Payload 
```c
#include <ps4.h>
#include <stdbool.h>

ScePthread thread;
bool * daemonExited;
bool * payloadLoaded;
void * (* mainFunction)(void *);

int64_t sceKernelDlsym(int64_t moduleHandle, const char * functionName, void * destFuncOffset) {
  return (int64_t) syscall(591, (void * ) moduleHandle, (void * ) functionName, destFuncOffset);
}

int loadModuleAndSymbols(const char * modulePath) {
  int prx_id = sceKernelLoadStartModule(modulePath, 0,
                                 NULL, 0, NULL, NULL);

  if (prx_id < 0) return -1;

  if (sceKernelDlsym(prx_id, "mainFunction", (void ** ) 
  &mainFunction) < 0 || mainFunction == NULL) return -1;

  if (sceKernelDlsym(prx_id, "payloadLoaded", (void ** ) 
  &payloadLoaded) < 0 || payloadLoaded == NULL) return -1;

  if (sceKernelDlsym(prx_id, "daemonExited", (void ** ) 
  &daemonExited) < 0 || daemonExited == NULL) return -1;
  *daemonExited = false; *payloadLoaded = true; return 0;
}

int _main(void) {
  if (loadModuleAndSymbols("/data/example.sprx") < 0)
  return -1; // can be a .PRX filetype but must be signed.

  if (scePthreadCreate(&thread, NULL, mainFunction,
            NULL, "ExampleThread") != 0) return -1;

  scePthreadJoin(thread, NULL);

  for (;;) {
    if (*daemonExited) {
      scePthreadDetach(thread);
      break; } sceKernelUsleep(1000000);
  } return 0;
}
```
<br>

## Example of SPRX 
```c
bool daemonExited  = false;
bool payloadLoaded = false;

void ThreadLoop() {
    while (!daemonExited) {
      printf("[DAEMON] Thread Looping...");
      sceKernelUsleep(1000000); continue;
    }

    // do any clean up work here if needed...
    // you'll need to handle this on your own,
    // that being a way to set daemonExited.
}

extern "C" void mainFunction() {
  if (payloadLoaded) ThreadLoop();
}
```
<br>

