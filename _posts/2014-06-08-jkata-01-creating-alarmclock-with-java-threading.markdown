---
layout: post
title: "JKata-01 Creating AlarmClock with Java Threading"
date: 2014-06-08 20:38:57 +1000
comments: true
author: Nam Nguyen
categories: [Java, Kata]
---

Applying TDD in Java with JUnit is interesting topic, in this blog I will creating a small application to demonstrate basic Java threading concepts and TDD
<!-- more -->

##Problem:

1. Create an AlarmClock class. A client can submit a single event and corresponding time. When the time passes, the AlarmClock sends an alarm to the client.
2. Eliminate the wait loop in AlarmClockTest (from the previous exercise) by using wait and notify
3. Modify the alarm clock to support multiple alarms (if it doesn't already). Then add the ability to cancel an alarm by name. Write a test that demonstrates the ability to cancel the alarm. Then analyze your code and note the potential for synchronization issues. Introduce a pause in the production code to force the issue and get your test to fail. Finally, introduce synchronization blocks or modify methods to be synchronized as necessary to fix the problem.
4. Change the run method in AlarmClock to use a timer to check for alarms every half-second.

##Solution:

First, create an Unit test class, this contains two test case which test functionality of AlarmClock class

* testAddEvent() method will test alarm clock ability to maintain list of alarm event
* testSendAlarm() method will test alarm clock by sending three events and attach a listener to receive event from the clock
* waitForEventPass() will wait until we achieve all alerts sent

``` java .AlarmClockTest.java
package jkata02.test;

import static org.junit.Assert.*;

import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.ArrayList;
import java.util.Calendar;
import java.util.Date;
import java.util.GregorianCalendar;
import java.util.List;

import jkata02.AlarmClock;
import jkata02.AlertEvent;
import jkata02.AlertListener;

import org.junit.After;
import org.junit.Before;
import org.junit.Ignore;
import org.junit.Test;

public class AlarmClockTest {

	private int alertedEvents = 0;
	private SimpleDateFormat sdf = new SimpleDateFormat("yyyy/MM/dd HH:mm:ss");
	@Before
	public void setupBefore() {}

	@After
	public void teardownAfter() {}

	@Test
	public void testAddEvent() throws Exception {
		AlarmClock clock = new AlarmClock();
		AlertEvent event1, event2, event3, event4;
		String time = "2014/06/06 00:00:00";
		event1 = new AlertEvent("cook", addSecondToDate(getSampleDate(time), 10));
		event2 = new AlertEvent("bath", addSecondToDate(getSampleDate(time), -10));
		event3 = new AlertEvent("run", addSecondToDate(getSampleDate(time), 20));
		event4 = new AlertEvent("sleep", addSecondToDate(getSampleDate(time), 15));

		clock.addEventToQueue(event1);
		clock.addEventToQueue(event2);
		clock.addEventToQueue(event3);
		clock.addEventToQueue(event4);

		// we make sure all event must be in the right order in the queue
		assertEquals(event2.getEventName(), clock.getFirstEvent().getEventName());
		assertEquals(event3.getEventName(), clock.getLastEvent().getEventName());
		List<AlertEvent> allEvents = clock.getAllCurrentEvents();
		String action = "";
		for(AlertEvent event: allEvents) {
			action += event.getEventName();
		}
		assertEquals("bathcooksleeprun", action);
	}

	@Test
	public void testSendAlarm() throws Exception {
		// list of event as input
		List<AlertEvent> events = new ArrayList<AlertEvent>();
		events.add(new AlertEvent("clean house", getCurrentTimePlusSecond(20)));
		events.add(new AlertEvent("cook dinner", getCurrentTimePlusSecond(30)));
		events.add(new AlertEvent("exercise", getCurrentTimePlusSecond(40)));

		// add listener to update clients alerts from AlarmClock
		AlertListener listener = new AlertListener() {
			public void onAlert(AlertEvent event) {
				System.out.println("Warning....It is time to " + event.getEventName() + " : " + sdf.format(event.getAlertTime().getTime()));
				alertedEvents++;
			}
		};

		// init clock with listener, start alarm clock thread
		AlarmClock clock = new AlarmClock(listener);
		clock.startClock();

		// add events to queue
		clock.addEvents(events);

		// wait for all event passed
		waitForEventPass(events.size());
		clock.stopClock();
	}


	// make this test wait for event to be passed
	private void waitForEventPass(int numberOfEvents) {
		while(alertedEvents < numberOfEvents) {
			try {
				Thread.sleep(1000);
			}
			catch(InterruptedException e) {	}
		}
	}

	private Calendar getCurrentTimePlusSecond(int seconds) {
		GregorianCalendar calendar = new GregorianCalendar();
		calendar.add(Calendar.SECOND, seconds);
		return calendar;
	}

	private Calendar addSecondToDate(Calendar date, int seconds) {
		date.add(Calendar.SECOND, seconds);
		return date;
	}

	/**
	 * @param dateString in format "yyyy/MM/dd HH:mm:ss"
	 * @return a Calendar object
	 * @throws ParseException
	 */
	private Calendar getSampleDate(String dateString) throws ParseException {
		GregorianCalendar calendar = new GregorianCalendar();
		calendar.setTime(sdf.parse(dateString));
		return calendar;
	}
}

```

A simple AlarmClock as follow:

Class AlertEvent is event object
```java AlertEvent.java
package jkata02;

import java.util.Calendar;

public class AlertEvent {
	private Calendar alertTime;
	private String eventName;
	public AlertEvent(String eventName, Calendar time) {
		this.setEventName(eventName);
		this.setAlertTime(time);
	}
	public Calendar getAlertTime() {
		return alertTime;
	}
	public void setAlertTime(Calendar alertTime) {
		this.alertTime = alertTime;
	}
	public String getEventName() {
		return eventName;
	}
	public void setEventName(String eventName) {
		this.eventName = eventName;
	}
}
```

Class AlertListener interface will be implemented by any client want to be notified by the clock via callback function onAlert
```java AlertListener.java
package jkata02;

public interface AlertListener {
	public void onAlert(AlertEvent event);
}
```

Main class clock extends Thread to mimic behaviour of a clock, some points to remember:

* eventQueue utilises Collections.synchronizedList() to protect the list from multiple thread access, this is good practice to follow
* use sorted LinkedList to support fast insert while still allow to retrieve list of passed event efficiently


```java AlarmClock.java
package jkata02;

import java.util.Collections;
import java.util.GregorianCalendar;
import java.util.LinkedList;
import java.util.List;
import java.util.ListIterator;

public class AlarmClock extends Thread {

	// synchronized list
	private List<AlertEvent> eventQueue = Collections
			.synchronizedList(new LinkedList<AlertEvent>());
	private AlertListener listener;
	private boolean clockRunning = true;

	public AlarmClock(AlertListener listener) {
		this.setListener(listener);
	}

	public AlarmClock() {}

	public void startClock() {
		this.start();
	}
	public void stopClock() {
		this.clockRunning = false;
	}

	public void addEvents(List<AlertEvent> events) {
		// TODO Auto-generated method stub
		for (AlertEvent alertEvent : events) {
			addEventToQueue(alertEvent);
		}
	}

	@Override
	public void run() {
		while (clockRunning) {
			// tic toc tic toc as clock run
			try {
				Thread.sleep(1000);
				System.out.println("tic");
			} catch (InterruptedException e) {
				// TODO Auto-generated catch block
				e.printStackTrace();
			}
			// check if any event need to be alerted
			if (!eventQueue.isEmpty()) {
				List<AlertEvent> passedEvents = getPassedEvent();

				// alert to clients
				alertClients(passedEvents);

				// remove passed event from queue
				eventQueue.removeAll(passedEvents);
			}
			Thread.yield();
		}
	}

	/**
	 * insert event to queue (linkedlist) and make sure it in right time order (for optimal searching event
	 * @param event
	 */
	public void addEventToQueue(AlertEvent event) {
		List<AlertEvent> list = this.eventQueue;
		if(list.size() == 0) list.add(event);
		else {
			ListIterator<AlertEvent> iterator = list.listIterator();
			do {
				AlertEvent e = iterator.next();
				if(event.getAlertTime().getTimeInMillis() <= e.getAlertTime().getTimeInMillis()) {
					if(list.indexOf(e) >= 0) list.add(list.indexOf(e), event);
					return;
				}
			}
			while(iterator.hasNext());
			list.add(event);
		}
	}

	/**
	 * alert clients
	 *
	 * @param passedEvents
	 */
	private void alertClients(List<AlertEvent> passedEvents) {
		// TODO Auto-generated method stub
		for (AlertEvent alertEvent : passedEvents) {
			this.getListener().onAlert(alertEvent);
		}
	}

	/**
	 * @return all passed events in the eventQueue
	 */
	private List<AlertEvent> getPassedEvent() {
		// TODO Auto-generated method stub
		GregorianCalendar calendar = new GregorianCalendar();
		long current = calendar.getTimeInMillis();
		ListIterator<AlertEvent> iterator = eventQueue.listIterator();
		List<AlertEvent> passedEvents = new LinkedList<AlertEvent>();
		while(iterator.hasNext()) {
			AlertEvent e = iterator.next();
			if(e.getAlertTime().getTimeInMillis() <= current)
				passedEvents.add(e);
		}
		return passedEvents;
	}

	private AlertListener getListener() {
		return listener;
	}

	private void setListener(AlertListener listener) {
		this.listener = listener;
	}

	public AlertEvent getLastEvent() {
		// TODO Auto-generated method stub
		if(this.eventQueue.size() > 0)
			return this.eventQueue.get(this.eventQueue.size() - 1);
		else
			return null;
	}

	public AlertEvent getFirstEvent() {
		if (this.eventQueue.size() > 0)
			return this.eventQueue.get(0);
		else
			return null;
	}

	public List<AlertEvent> getAllCurrentEvents() {
		return this.eventQueue;
	}
}
```

Done, run the unit test and you can see how it works

I will address problem 2 next time


