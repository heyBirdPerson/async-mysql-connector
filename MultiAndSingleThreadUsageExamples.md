
```
 CREATE TABLE `test` (
  `id` int(11) NOT NULL,
  `updated` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
  `status` int(11) NOT NULL,
  `content` varchar(255) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=MEMORY DEFAULT CHARSET=latin1
```

with 1M random rows

# Thread Safe Multiplexer Wrapper #

```
package async.test;

import java.io.IOException;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
import java.util.concurrent.ArrayBlockingQueue;
import java.util.concurrent.DelayQueue;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.TimeUnit;




import org.async.jdbc.SuccessCallback;
import org.async.mysql.MysqlConnection;
import org.async.mysql.protocol.packets.OK;
import org.async.net.Multiplexer;

public class ThreadSafeWrapper {
	public interface Query {
		void doInConnection(MysqlConnection con);
		
	}
	final ArrayBlockingQueue<Query> q=new ArrayBlockingQueue<Query>(64);
	ExecutorService pool;
	List<Multiplexer> mpxs=new ArrayList<Multiplexer>();
	public  ThreadSafeWrapper(int mpx) throws IOException {
		pool = Executors.newFixedThreadPool(mpx);
		
		for(int i=0;i<mpx;i++) {
			final Multiplexer multiplexer = new Multiplexer();
			mpxs.add(multiplexer);
			pool.execute(new Runnable() {
				@Override
				public void run() {
					while(true) {
						try {
							multiplexer.select();
							
							
								final Query e = q.poll();
								if(e!=null) {
									final MysqlConnection[] con=new MysqlConnection[1];
									con[0]=new MysqlConnection("localhost", 3306, "root", "sa", "asynctest", multiplexer.getSelector(), 
											new SuccessCallback() {
												
												@Override
												public void onError(SQLException e) {
													try {
														con[0].close();
													} catch (SQLException e1) {
													
														e1.printStackTrace();
													}
													
												}
												
												@Override
												public void onSuccess(OK ok) {
													e.doInConnection(con[0]);
													
													
												}
											}
											);
								}
							
								
							
						} catch (IOException e) {
							e.printStackTrace();
							return;
						}
					}
					
				}
			});
		}
		
	}
	public void q(Query query) throws InterruptedException {
		q.put(query);
		for(Multiplexer mpx:mpxs) {
			mpx.getSelector().wakeup();
		}
		
	}
	
	public void stop() {
		pool.shutdown();
	}
	
}
```

# and the tests #


```
package async.test;

import java.sql.SQLException;

import org.async.jdbc.AsyncConnection;
import org.async.jdbc.Connection;
import org.async.jdbc.PreparedQuery;
import org.async.jdbc.PreparedStatement;
import org.async.jdbc.ResultSet;
import org.async.jdbc.ResultSetCallback;
import org.async.jdbc.SuccessCallback;
import org.async.mysql.MysqlConnection;
import org.async.mysql.protocol.packets.OK;
import org.async.net.Multiplexer;
import org.junit.Test;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;
import java.util.Random;
import java.util.concurrent.atomic.AtomicBoolean;

public class TestAsync2 {

	private final int TEST_DURATION = 120000;

	

	@Test
	public void singleThreadTest() throws IOException {
	
		final Random random = new Random();
		Multiplexer mpx = new Multiplexer();
		final List<MysqlConnection> cons = new ArrayList<MysqlConnection>();
		final AtomicBoolean stop=new AtomicBoolean(false);
		for (int i = 0; i < 32; i++) {
			final int idx=i;
			cons.add(new MysqlConnection("localhost", 3306,
					"root", "sa", "asynctest", mpx.getSelector(),
					new SuccessCallback() {

						@Override
						public void onError(SQLException e) {
							e.printStackTrace();

						}

						@Override
						public void onSuccess(OK ok) {
							try {
							    final PreparedStatement ps = cons.get(idx).prepareStatement("select SQL_NO_CACHE * from test where id=?");

								final PreparedQuery query = new PreparedQuery() {

									@Override
									public void query(PreparedStatement pstmt)
											throws SQLException {
										pstmt.setInteger(1,
												random.nextInt(1000000));

									}
								};
								final ResultSetCallback rsc = new ResultSetCallback() {
									int a = 0;

									@Override
									public void onError(SQLException e) {
										e.printStackTrace();

									}

									@Override
									public void onResultSet(ResultSet rs) {
										while (rs.hasNext()) {
											rs.next();
											int id = rs.getInteger(1);
											java.sql.Timestamp date = rs
													.getTimestamp(2);
											int status = rs.getInteger(3);
											String text = rs.getString(4);
										}
										
										try {
											if(!stop.get()) {
												ps.executeQuery(query, this);
											} else {
												cons.get(idx).close();
											}
										} catch (SQLException e) {

											e.printStackTrace();
										}
										a++;

									}
								};
								ps.executeQuery(query, rsc);
							} catch (SQLException e1) {
								e1.printStackTrace();
							}
						}
					}));
			}
			
			long start=System.currentTimeMillis();
			while (System.currentTimeMillis()-start<TEST_DURATION) {
				mpx.select();
				
			}
			
			
			
			stop.set(true);
			for (int i=0;i<10;i++) {
				mpx.select(100);
				
			}
		}

	@Test
	public void multiThreadTest() throws IOException, InterruptedException {
		final ThreadSafeWrapper wrapper = new ThreadSafeWrapper(8);
		
		
		final Random random = new Random();
		
	
			
			
			for(int i=0;i<32;i++) {
				new Thread(new Runnable() {

					@Override
					public void run() {
						try {
							wrapper.q(new ThreadSafeWrapper.Query() {
								PreparedStatement ps;
								final PreparedQuery query = new PreparedQuery() {

									@Override
									public void query(PreparedStatement pstmt)
											throws SQLException {
										pstmt.setInteger(1,
												random.nextInt(1000000));

									}
								};
								final ResultSetCallback rsc = new ResultSetCallback() {
									int a = 0;

									@Override
									public void onError(SQLException e) {
										e.printStackTrace();

									}

									@Override
									public void onResultSet(ResultSet rs) {
										while (rs.hasNext()) {
											rs.next();
											int id = rs.getInteger(1);
											java.sql.Timestamp date = rs
													.getTimestamp(2);
											int status = rs.getInteger(3);
											String text = rs.getString(4);
										}
										
										try {
											ps.executeQuery(query, this);
										} catch (SQLException e) {

											e.printStackTrace();
										}
										a++;

									}
								};
								@Override
								public void doInConnection(MysqlConnection con) {
									try {
										if(ps==null) ps=con.prepareStatement("select SQL_NO_CACHE * from test where id=?");
										ps.executeQuery(query, rsc);
									} catch (SQLException e) {
										e.printStackTrace();
									}
									
									
								}
								
							});
						} catch (InterruptedException e) {
							return;
							
						}
						
					}
					
				}).start();
				
				
			}
			
			Thread.sleep(TEST_DURATION);
			
			
			wrapper.stop();
			

		}


}

```