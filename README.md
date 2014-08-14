lttng-dynamic-instrumentation
=============================
#Current status
In the current state of the project, the main feature is to ability to run lttng trace command. Using the '-f' argument, you can specify which function on the application you want traced.
<pre>
$> lttng trace -f montreal ./app
lttng trace -f montreal -n foo ../dyninst-sandbox/suspect                                                                                                       
Trace directory: net://localhost/host/gamma/auto-20140814-125539

[12:55:42.205438197] (+?.?????????) gamma dyn-tp:foo_entry: { cpu_id = 2 }, { vpid = 7763 }, { a = 24, _b_length = 33, b = "Life is like a box of chocolates", c = 112 }
[12:55:42.205493582] (+0.000055385) gamma dyn-tp:foo_exit: { cpu_id = 2 }, { vpid = 7763 }, { }
[12:55:43.205653872] (+1.000160290) gamma dyn-tp:foo_entry: { cpu_id = 2 }, { vpid = 7763 }, { a = 24, _b_length = 33, b = "Life is like a box of chocolates", c = 112 }
[12:55:43.205684182] (+0.000030310) gamma dyn-tp:foo_exit: { cpu_id = 2 }, { vpid = 7763 }, { }
[12:55:44.205854602] (+1.000170420) gamma dyn-tp:foo_entry: { cpu_id = 2 }, { vpid = 7763 }, { a = 24, _b_length = 33, b = "Life is like a box of chocolates", c = 112 }
[12:55:44.205885462] (+0.000030860) gamma dyn-tp:foo_exit: { cpu_id = 2 }, { vpid = 7763 }, { }
^CWaiting for data availability
Tracing stopped for session auto-20140814-125539
Session auto-20140814-125539 destroyed
</pre>

#Working
1. Instrument function entry
2. Retrieve function parameter type
3. Retrieve function parameter name
4. Retrieve function parameter value
5. Instrument function exit

#Not working

1. Inserting Multiple dynamic tracepoints
2. Removing dynamic tracepoints
3. Disabling dynamic tracepoints
4. Probe instrumentation
5. Detaching and reattaching to the same process. Dyninst doesn't keep any mapping of the memory address used in previous attach. So reattaching and writing to tracee address space will overwrite previously copied data.
6. Tracing floating point parameter

#Known bugs
1. Trigerring tracepoint if the session has not started.

#TODO
1. Add probe instrumenation (easy)
2. Probe regristration moved to the session daemon(hard)

#How it is done?
##Overview
1. When the dynamic loader loads the liblttng-ust.so library in the APP's address space, the constructor of the lib is called. This is at this point that the app is registered to the session daemon. Information on the app is sent to the session daemon, such as the process name and process id.
2. The session daemon receives the name of the process and se that it matches with a dynamic tracepoint currently enabled in the current session. It uses Dyninst to attach to the process. This put the app in the stopped status.
3. At this point, the session is attached to the process and has access to the whole app's address space. Using Dyninst, the session daemon retrieves the targeted function and its parameter's name and type through the debugging symbols. Using this tracepoint and probe structres are dynamically generated and copy into the app's address space.
4. To register a tracepoint, the tracepoint_register and lttng_probe_register functions must be called from the app so we hook calls to those functions to an empty function in the app. This function is call lttng_ust_fake_function and has the only purpose of being hooked on for the registration of tracepoints. This step will discuss in details in the next section.
5. At step 5, the session daemon add instrumentation on the function entry to record an event. The recording of an event is made in three main steps that are initializing a context in the ringbuffer, write the data for each field, and commit the event. Those three steps must be done in this order each time the event is triggered. I hook those functions call to the targeted function's entry.
6. All the needed changes to the app are done. The session daemon can now continue the execution of the app.
7. Through a unix socket, the session daemon sends a command to call the lttng_ust_fake_function. This call will trigger the calls that were added in step 4.
8. The tracepoint_register function is called.
9. The lttng_probe_register function is called.
10. After sending any other enabled static tracepoints, the session daemon sends a "Registration done" command that specify that the app can exit the liblttng-ust.so's constructor and eventually start executing the main function.
![Alt text](img/lttng-di.png "High level diagram")

##Tracepoint registration
This tracepoint registration process is made of three main actions. First, there is the tracepoint registration followed by the probe registration and finally setting the registered flag to TRUE. The registered flag must be set to TRUE in order for the tracepoint recording sequence to be enabled(see next section). This call sequence was hooked to the lttng_ust_fake_function at step #4 of the previous section.
###Pseudo-code
<pre>

bool isTpRegistered = false;

registerTP();

registerProbe();

completeRegistration(){
	isTpRegistered = true;
}

lttng_ust_fake_function{
	registerTP();
	registerProbe();
	completeRegistration();
}

</pre>

##Event recording
To trace an event four main things need to be done. Firstly, we have to compute the sum of the length of each field on the event. This sum is not static since some CTF fields are of variable length(e.g. ctf_sequence). Secondly, we have to initialize a ringbuffer context for this tracepoint and its length. Thirdly, we write each field in the ringbuffer. Finally, we commit the context. Once a context is commited it can be consumed by the consumer daemon and in turn write to a file. Notice that writing and commiting the context are only done if the context is initialized.

N.B. The following pseudocode makes it look like function calls are added inside the target function. I doubt that it's how Dyninst really does it but it's simpler for my explanations.
###Pseudo-code
<pre>
bool isCtxReady = false;
int event_len = 0;
Context ctx;

void addIntLen(){
	event_len += sizeof(int);
}

void addCharLen(){
	event_len += sizeof(char);
}

void InitCtx(){
	//We need event length to reserve
	//space in the context ringbuffer
	if(isTpRegistered){
		init(ctx, event_len);
		IsCtxReady = true;
	}
}

void writeIntField(int value){
	if(!isCtxReady)
		return;
	write(ctx, &value);	
}
void writeCharField(char value){
	if(!isCtxReady)
		return;
	write(ctx, &value);	
}

void commitEvent(){
	if(!isCtxReady)
		return;
	commit(ctx);
}


void interestingFunction(int a, char b){
	addIntLen();
	addCharLen();
	InitCtx();
	writeIntField(a);
	writeCharField(b);
	commitEvent();
	//... original function
}
</pre>

##Dyninst
Dyninst is a powerful library that can retreive information on, modify address space, and infect code in a running process. This prototype uses dyninst to copy data structures in the tracee address space and to add calls to functions for the tracepoint registration and event recording.

website: http://www.dyninst.org/

git: http://git.dyninst.org/ 

##Structures
N.B. In the following explanation, I will use a tracepoint named prov:name and has one integer(32 bits) field and char(8 bits) field.

In order to register a tracepoint, at least four structures must be constructed. 
The tracepoint structure describes its name and signature. Only those two fields are used in this implementation of dynamic instrumentation. The name and signature fields which are respectively "prov:name" and "name" in our example.

<pre>
struct tracepoint {
	const char *name;
	int state;
	struct tracepoint_probe *probes;
	int *tracepoint_provider_ref;
	const char *signature;
	char padding[TRACEPOINT_PADDING];
};
</pre>

The following structure describes the tracepoints related to a provider. The "provider" field in this case would be set to "prov". The nr_events is set to the number of events for this provider. The field event_desc points to an array of nr_events event descriptions.

<pre>
struct lttng_probe_desc {
	const char *provider;
	const struct lttng_event_desc **event_desc;
	unsigned int nr_events;
	struct cds_list_head head;		/* chain registered probes */
	struct cds_list_head lazy_init_head;
	int lazy;				/* lazy registration */
	uint32_t major;
	uint32_t minor;
	enum lttng_probe_type type;
	char padding[LTTNG_UST_PROBE_DESC_PADDING];
};
</pre>

This structure represents a tracepoint. At the moment, we are interested in its name, its number of field and to its array of field descriptions.
<pre>
struct lttng_event_desc {
	const char *name;
	void (*probe_callback)(void);
	const struct lttng_event_ctx *ctx;	/* context */
	const struct lttng_event_field *fields;	/* event payload */
	unsigned int nr_fields;
	const int **loglevel;
	const char *signature;	/* Argument types/names received */
	union {
		struct {
			const char **model_emf_uri;
		} ext;
		char padding[LTTNG_UST_EVENT_DESC_PADDING];
	} u;
};
</pre>
This structure describe a field. In this implementation, the name and the type are useful. In our exmaple, we would have a field of type 32 bits and 8 bits integer.
<pre>
struct lttng_event_di_field {
	const char *name;
	struct lttng_type type;
	unsigned int nowrite;	/* do not write into trace */
	char padding[LTTNG_UST_EVENT_FIELD_PADDING];
};
</pre>







