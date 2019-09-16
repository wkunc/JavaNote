# Transport
是一个abstract类, 但是却有static方法来直接发送Message.

public static void send(Message msg, String user, String password)
public static void send(Message msg,Address[] addresss, String user, String password)

```java
private static void send0(Message msg, Address[] address,
    String user, String password) {
    if (addresses == null || addreses.length == 0)
        throw new SendFailedException("No recipient addresses");

    Map<String, List<Address>> protocols = new HashMap<>();

	// Lists of addresses
	List<Address> invalid = new ArrayList<>();
	List<Address> validSent = new ArrayList<>();
	List<Address> validUnsent = new ArrayList<>();

	for (int i = 0; i < addresses.length; i++) {
	    // is this address type already in the map?
	    if (protocols.containsKey(addresses[i].getType())) {
            List<Address> v = protocols.get(addresses[i].getType());
            v.add(addresses[i]);
	    } else {
            // need to add a new protocol
            List<Address> w = new ArrayList<>();
            w.add(addresses[i]);
            protocols.put(addresses[i].getType(), w);
	    }
	}

	int dsize = protocols.size();
	if (dsize == 0)
	    throw new SendFailedException("No recipient addresses");

	Session s = (msg.session != null) ? msg.session :
		     Session.getDefaultInstance(System.getProperties(), null);
	Transport transport;

	/*
	 * Optimize the case of a single protocol.
	 */
	if (dsize == 1) {
	    transport = s.getTransport(addresses[0]);
	    try {
            if (user != null)
                transport.connect(user, password);
            else
                transport.connect();
            transport.sendMessage(msg, addresses);
	    } finally {
            transport.close();
	    }
	    return;
	}

	/*
	 * More than one protocol.  Have to do them one at a time
	 * and collect addresses and chain exceptions.
	 */
	MessagingException chainedEx = null;
	boolean sendFailed = false;

	for(List<Address> v : protocols.values()) {
	    Address[] protaddresses = new Address[v.size()];
	    v.toArray(protaddresses);

	    // Get a Transport that can handle this address type.
	    if ((transport = s.getTransport(protaddresses[0])) == null) {
		// Could not find an appropriate Transport ..
		// Mark these addresses invalid.
		for (int j = 0; j < protaddresses.length; j++)
		    invalid.add(protaddresses[j]);
		continue;
	    }
	    try {
		transport.connect();
		transport.sendMessage(msg, protaddresses);
	    } catch (SendFailedException sex) {
		sendFailed = true;
		// chain the exception we're catching to any previous ones
		if (chainedEx == null)
		    chainedEx = sex;
		else
		    chainedEx.setNextException(sex);

		// retrieve invalid addresses
		Address[] a = sex.getInvalidAddresses();
		if (a != null)
		    for (int j = 0; j < a.length; j++) 
			invalid.add(a[j]);

		// retrieve validSent addresses
		a = sex.getValidSentAddresses();
		if (a != null)
		    for (int k = 0; k < a.length; k++) 
			validSent.add(a[k]);

		// retrieve validUnsent addresses
		Address[] c = sex.getValidUnsentAddresses();
		if (c != null)
		    for (int l = 0; l < c.length; l++) 
			validUnsent.add(c[l]);
	    } catch (MessagingException mex) {
		sendFailed = true;
		// chain the exception we're catching to any previous ones
		if (chainedEx == null)
		    chainedEx = mex;
		else
		    chainedEx.setNextException(mex);
	    } finally {
		transport.close();
	    }
	}

	// done with all protocols. throw exception if something failed
	if (sendFailed || invalid.size() != 0 || validUnsent.size() != 0) { 
	    Address[] a = null, b = null, c = null;

	    // copy address lists into arrays
	    if (validSent.size() > 0) {
		a = new Address[validSent.size()];
		validSent.toArray(a);
	    }
	    if (validUnsent.size() > 0) {
		b = new Address[validUnsent.size()];
		validUnsent.toArray(b);
	    }
	    if (invalid.size() > 0) {
		c = new Address[invalid.size()];
		invalid.toArray(c);
	    }
	    throw new SendFailedException("Sending failed", chainedEx, 
					  a, b, c);
	}
}
```
